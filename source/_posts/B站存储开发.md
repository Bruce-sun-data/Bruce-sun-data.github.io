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

将数据写入nvme的三种方式

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

要实现一个内核的模块

并且需要写一个内核模块的Makefile，进行编译



### 用SPDK



## SPDK作用

## 内核文件系统与SPDK文件系统

## 实现媲美 ext4读写的文件系统

## bio与nvme落盘









