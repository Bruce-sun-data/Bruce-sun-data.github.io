---
title: SPDK使用vhost
date: 2024-04-11 21:10:24
tags:
typora-root-url: ..
categories:
    - 存储研究
---

# SPDK使用vhost

## SPDK vhost工作机制

![image-20240416101340921](/images/SPDK使用vhost/image-20240416101340921.png)

SPDK vhost以进程的形式在本地计算机提供存储服务，

- 虚拟机中的存储前端驱动：根据具体使用的协议不同而不同，可能为 virtio blk，virtio scsi，或直接使用原生的 NVMe driver，主要负责接收 IO 请求，并将请求保存到 Ring Buffer 中，等待后端的处理。另一方面，前端驱动还需要处理后端的完成通知；

- 共享的 Ring Buffer：虚拟机中的前端驱动与宿主中的存储后端共享的内存，用来在两者之间传输数据，命令和执行结果；
- SPDK vhost 后端：虚拟机存储的后端实现，真正处理 IO 请求的核心逻辑，通过用户态的存储驱动实现高效的 IO 操作。

## 安装QEMU

通过官网下载了qemu-7.2.0(如果出错了了就换版本安装)

```shell
wget https://download.qemu.org/qemu-7.2.0.tar.xz
#安装依赖包
sudo apt-get install git libglib2.0-dev libfdt-dev libpixman-1-dev zlib1g-dev
#编译安装
tar xvJf qemu-5.0.0.tar.xz
cd qemu-7.2.0
mkdir build
cd build
../configure
make
```

make需要很久，要等一下。

然后在build目录下运行`qemu-system-x86_64 --version` 显示qemu版本

```
QEMU emulator version 7.2.0
Copyright (c) 2003-2022 Fabrice Bellard and the QEMU Project developers
```

然后根据[SPDK官网](https://spdk.io/doc/vhost.html)的提示运行

```
# qemu-system-x86_64 -device vhost-user-scsi-pci,help
# qemu-system-x86_64 -device vhost-user-blk-pci,help
```

这两个命令会返回命令的相关参数。如果都有输出就对了，否则继续换版本。

## QEMU使用

必须指定运行的CPU和基础镜像。可以在https://cloud-images.ubuntu.com/bionic/20230607/中下载ubuntu18.04的.img文件，然后通过qemu-img转化为.qcow2，然后提供给qemu作为磁盘映像。

[qemu-img命令详解](https://blog.csdn.net/c_base_jin/article/details/104126542)

在服务器中安装虚拟机(没有图形界面安装没有图形界面的虚拟机)https://blog.csdn.net/qq_32305507/article/details/119976049

j教程不管用

最终还是决定了使用图形化界面安装QEMU

### 在服务器中安装vnc，实现没有显示器的远程访问

借鉴这一篇博客https://help.aliyun.com/zh/simple-application-server/use-cases/use-vnc-to-build-guis-on-ubuntu-18-04-and-20-04#21e09061d7w5p

注意，在客户端使用VNC Viewer的时候，关闭翻墙软件

如果再出现黑屏，重启机器

### 安装QEMU虚拟机

首先准备了一个1TB的机械硬盘。然后分区300GB，进行挂载。

在该机械硬盘中生成.qcow2文件作为虚拟磁盘，会作为虚拟机的系统盘。

创建30G的虚拟磁盘

```shell
qemu-img create -f qcow2 test.qcow2 30G
```

使用cdrom(ubuntu的iso的镜像文件)安装虚拟机

```shell
sudo ../QEMU/qemu-7.2.0/build/qemu-system-x86_64  -hda ubuntu-server1.qcow2 -cdrom ./ubuntu-18.04.6-live-server-amd64.iso -boot d -m 2048 
```

然后直接就会跳出来QEMU的界面，然后先不设置网络，一切都点ok

安装完之后直接退出界面，然后使用如下命令直接可以启动。

```shell
sudo ../QEMU/qemu-7.2.0/build/qemu-system-x86_64 -hda test.qcow2 -boot c -m 1024 
```



### 给虚拟机配置网络

其中需要==设置网络参数==详见https://blog.csdn.net/v6543210/article/details/117668461

在使用`ip a`命令的时候发现有`virbr0`,直接使用NAT模式

直接使用virbr0可以看https://blog.csdn.net/liukuan73/article/details/49231785这篇文章。但是看不明白。

还是使用桥接模式，但是还是不太管用。为了能够正确使用，对每一种网络连接方法进行深入了解

QEMU提供了4种网络通信的方式

[QEMU关于网络配置的文档](https://wiki.qemu.org/Documentation/Networking)

1. User mode stack：

   如果不指定nic，那么默认的就是user模式。以下三个命令等效

   ```
   qemu -m 256 -hda disk.img &
   
   qemu -m 256 -hda disk.img -net nic -net user & #使用 -net user 必须同 -net nic配合
   
   qemu-system-i386 -m 256 -hda disk.img -netdev user,id=network0 -device e1000,netdev=network0,mac=52:54:00:12:34:56 &
   ```

   在该模式中，

   

2. 



### 设置nographic启动







## 参考文献

https://rootw.github.io/2018/05/SPDK-all/   

[浅析SPDK技术：vhost](https://blog.csdn.net/anyegongjuezjd/article/details/136123555)
