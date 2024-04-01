---
title: B站存储开发
date: 2024-03-12 17:07:37
tags:
typora-root-url: ..
categories:
    - 存储研究
---

# 存储体系介绍

从事存储领域的工作可搜索词汇：spdk,ceph,rocksdb,nvme

## 自上而下的存储栈

![image-20240312210534587](./images/B站存储开发/image-20240312210534587.png)

# 存储开发三部曲，内核文件系统实现

## 如何自己写一个文件系统

因为不能直接用read或者write命令往磁盘中写内容，所以需要文件系统

### 方法1  构建一个磁盘

```shell
dd if=/dev/zero of=test.img bs=1M count=50
```

IMG是镜像文件。test.img和普通磁盘一样。

Linux dd 命令用于读取、转换并输出数据。

![image-20240312210711757](./images/B站存储开发/image-20240312210711757.png)

先生成50MB的img

```shell
mkfs.ext4 test.img
```

对test.img进行EXT4格式化操作。

```
sudo mount -t ext4 test.img ./mnt/
```

如果在mnt中写一个东西，那么这个东西就存储在了test.img中

### 方法2  创建一个磁盘

直接用盘创建一个

### 如何操作磁盘

如果增加了一块裸盘，可以直接向裸盘写数据，然后通过cat读出来。即不需要文件系统也可以操作。

### 实现文件系统三部曲

1. 落盘：写的数据能够写进盘里
2. 内核模块
3. 文件系统的内容

# 存储的三座大山，磁盘，内核文件系统，分布式文件系统

<img src="./images/B站存储开发/image-20240313204120783.png" alt="image-20240313204120783" style="zoom:67%;" />

按照数据不同的需求，可以使用不同的文件系统

<img src="./images/B站存储开发/image-20240313205244761.png" alt="image-20240313205244761" style="zoom:67%;" />

将数据写入nvme裸盘的三种方式(三种方式实现文件系统)

### 在应用层，用提供出来的接口操作nvme

直接对裸盘进行分区，不进行文件系统操作。然后使用命令`dd if=/dev/zero of=/dev/nvme1n1p1`将该盘数据清空。

   用下面的代码可以直接在应用层对没有文件系统的裸进行 操作

   ```c
   //nvme.c   gcc -o nvme nvme.c 
   #include<stdlib.h>
   #include<stdio.h>
   #include<string.h>
   #include<unistd.h>
   #include<fcntl.h>
   #include<sys/ioctl.h>
   #include<linux/nvme_ioctl.h>
   
   int main(){
   
   	int fd = open("/dev/nvme1n1p1",O_RDWR);
   	if(fd<0)
   		return -1;
   	char *buffer = malloc(4096);
   	if(!buffer){
   		perror("malloc\n");
   		return -1;
   	}
   // 先清空
   	memset(buffer,0,4096);
   
   	struct nvme_user_io io;
   	// 地址
   	io.addr = (__u64)buffer;
   	// 逻辑块地址
   	io.slba = 0;
   	// 连续写的块数
   	io.nblocks=1;
   	// 1是写
   	io.opcode=1;
   	strcpy(buffer,"ABCDEFGHIJKLMNOPQRSTUVWXYZ");
   	if(-1==ioctl(fd,NVME_IOCTL_SUBMIT_IO,&io)){
   		perror("ioctl\n");
   		close(fd);
   		free(buffer);
   	}
   	printf("write succ\n");
   	return 0;
   }
   ```

   使用`cat /dev/nvme1n1p1`可以查看写入内容

   因为之前给/dev/loop30/挂载的时候分配过文件系统，所以会显示乱码

### 在内核里读写nvme

需要写的文件不能用main，因为main是应用程序。

要实现一个内核的模块，在通过insmode和rmmode触发磁盘的读写操作

```c
#include<linux/bio.h>
#include<linux/module.h>
#include<linux/blkdev.h>
#include<linux/fs.h>
#include<linux/init.h>

#define DISK_NAME "/dev/nvme1n1p1"
struct block_device *bdev=NULL;
#define DISK_SECTOR_SIZE 512

static struct block_device *open_disk(const char *devname){
	struct block_device *bdev=NULL;
	bdev = blkdev_get_by_path(devname,FMODE_READ | FMODE_WRITE,NULL);
	if (IS_ERR(bdev))
	{
		printk("blkdev_get_by_path err\n");
		return NULL;
	}
	return bdev;
}

static int sun_nvme_write(struct block_device *bdev,char *buffer, int length){
	struct page *page;
// 该值一般是256
	int max_vecs = BIO_MAX_PAGES;
	
	// 初始化bio
	struct bio *bio = bio_alloc(GFP_NOIO,max_vecs);
	bio_set_dev(bio,bdev);
	bio->bi_opf = REQ_OP_WRITE;
	if (!bio)
		return -1;

	bio->bi_iter.bi_sector=0; //逻辑块地址
	
	page = alloc_page(GFP_KERNEL);
	if(!page)
		return -1;
	// page_address提供page的虚拟地址
	memcpy(page_address(page),buffer,length);
	
	bio_add_page(bio,page,512,0);
	submit_bio_wait(bio);
	__free_page(page);
	bio_put(bio);
	return 0;

}

static int sun_nvme_read(struct block_device *bdev,char *buffer, int length){

	struct page *page;
	void *str = NULL;
	int max_vecs = BIO_MAX_PAGES;
	struct bio *bio = bio_alloc(GFP_NOIO,max_vecs);
	bio_set_dev(bio,bdev);
	bio->bi_opf = REQ_OP_READ;
	if (!bio)
		return -1;
	// 开始读的位置
	bio->bi_iter.bi_sector=0; //逻辑块地址
	
	page = alloc_page(GFP_KERNEL);
	if(!page)
		return -1;
	bio->bi_private=page;
	// memcpy(page_address(page),buffer,length);
	// bio_add_page(bio,virt_to_page(buffer),length,0);
	bio_add_page(bio,page,PAGE_SIZE,0);


	submit_bio_wait(bio);

	str = page_address(page);
	memcpy(buffer,str,length);


	__free_page(page);
	bio_put(bio);

	return 0;
}

// insmod kernel_nvme.ko
static int sun_nvme_init(void){
	printk("sun_nvme_init\n");

	char buffer[512]={0};
	bdev = open_disk(DISK_NAME);
	if(!bdev)
		return -1;
	memset(buffer,0,DISK_SECTOR_SIZE);

	strcpy(buffer,"========ABCDEFGHIJKLMNOPQRSTUVWXYZ");

	
	int k;
	// 写操作
	k = sun_nvme_write(bdev,buffer,strlen(buffer));
	printk(KERN_INFO "write succ %d\n",k);
	memset(buffer,0,DISK_SECTOR_SIZE);

	// 写成功了进行读
	sun_nvme_read(bdev,buffer,DISK_SECTOR_SIZE);
	printk(KERN_INFO "read succ\n");
	printk(KERN_INFO "buffer: %s\n",buffer);

	return 0;
}
// rmmod kernel_nvme.ko

static void sun_nvme_exit(void){
	printk(KERN_INFO "un_nvme_exit\n");


}

module_init(sun_nvme_init);
module_exit(sun_nvme_exit);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Your Name");
MODULE_DESCRIPTION("A simple example Linux module.");
MODULE_VERSION("0.1");
```

并且需要写一个内核模块的Makefile，进行编译

```shell
obj-m := kernel_nvme.o

KERNELBUILD := /lib/modules/$(shell uname -r)/build

default:
	make -C $(KERNELBUILD) M=$(shell pwd) modules
```





# SPDK

同一个host里，fio使用不同的引擎会有不同的结果(iops)

![image-20240315194810449](/images/B站存储开发/image-20240315194810449.png)

==psync==

是同步的读写，第二次写需要等第一次写返回才能写。每一次write是串行的

![image-20240315194857240](/images/B站存储开发/image-20240315194857240.png)



==libaio==

是异步的读写，在libaio这一层就返回了，liaio自己本身负责落盘。文件系统和**psync**是一样的

![image-20240315195047494](/images/B站存储开发/image-20240315195047494.png)

==spdk_bdev==

bdevice和nvme裸盘的区别

bdev更贴近于纯软件

![image-20240322152814191](/images/B站存储开发/image-20240322152814191.png)

blob通过函数调用bdev，bdev通过rpc调用数据到nvme

##  代码实现

在`/spdk/examples/`中创建文件夹和makefile文件，进行example的编写

在`spdk/build/examples`中运行对应的example

操作blob分为如下步骤

1. create_bdev:创建
2. blobstore_init
3. create_blob
4. open_blob
5. write_blob
6. read_blob
7. close_blob
8. destory_blob

基于以上八步，可以实现blob管理组件(文件系统)

## 内核文件系统与SPDK文件系统

## 实现媲美 ext4读写的文件系统

## bio与nvme落盘









