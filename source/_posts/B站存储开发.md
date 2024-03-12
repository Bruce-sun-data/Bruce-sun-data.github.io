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

![image-20240312210534587](/images/B站存储开发/image-20240312210534587.png)

# 存储开发三部曲，内核文件系统实现

## 如何自己写一个文件系统

因为不能直接用read或者write命令往磁盘中写内容，所以需要文件系统

### 方法1  构建一个磁盘

```shell
dd if=/dev/zero of=test.img bs=1M count=50
```

IMG是镜像文件。test.img和普通磁盘一样。

Linux dd 命令用于读取、转换并输出数据。

![image-20240312210711757](/images/B站存储开发/image-20240312210711757.png)

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



