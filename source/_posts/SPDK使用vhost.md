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

通过官网下载了qemu-5.0.0

```shell
wget https://download.qemu.org/qemu-5.0.0.tar.xz
#安装依赖包
sudo apt-get install git libglib2.0-dev libfdt-dev libpixman-1-dev zlib1g-dev
#编译安装
tar xvJf qemu-5.0.0.tar.xz
cd qemu-5.0.0
mkdir build
cd build
../configure
make
```

make需要很久，要等一下

然后在build目录下运行`qemu-system-x86_64 --version` 出现报错

```shell
qemu-system-x86_64: error while loading shared libraries: libncursesw.so.6: cannot open shared object file: No such file or directory
```







## 参考文献

https://rootw.github.io/2018/05/SPDK-all/   

[浅析SPDK技术：vhost](https://blog.csdn.net/anyegongjuezjd/article/details/136123555)
