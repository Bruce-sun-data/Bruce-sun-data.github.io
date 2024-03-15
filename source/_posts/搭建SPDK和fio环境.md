---
title: 搭建SPDK和fio环境
date: 2024-03-15 20:26:22
tags:
typora-root-url: ..
categories:
    - 存储研究
---

# fio

## 安装

这里使用github的安装方式，方便管理

使用内网的github网站

```shell
git clone https://gitcode.com/axboe/fio.git
```

接着在fio的目录下运行下面的命令

```shell
sudo apt-get install libaio-dev
sudo apt-get install gcc
sudo ./configure
sudo make install
fio -v
```

可以看到fio的版本(这里的版本是`fio-3.36-38-g06c4`)

fio的官方使用手册https://fio.readthedocs.io/en/latest/fio_doc.html

## 测试

主要使用.fio文件进行测试

### 测试psync

.fio文件内容

```shell
[global]
thread=1
group_reporting=1
ioengine=psync
direct=1
verify=0
size=16384
time_based=1
rw=randwrite
runtime=10
bs=16K
iodepth=64
filename=/dev/nvme1n1p1
[test]
stonewall
description="variable bs"
bs=16K

```

测试命令

```shell
path_to_fio/fio u_path/psync.fio
```

测试结果

<img src="/images/搭建SPDK和fio环境/image-20240315210452247.png" alt="image-20240315210452247"  />

### 测试libaio

只需要把上面的`psync`换成`libaio`就可以了

![image-20240315210748428](/images/搭建SPDK和fio环境/image-20240315210748428.png)

效果有一点点提升

# SPDK

## 安装

