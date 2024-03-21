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

## 安装SPDK

先查看我们系统的环境

```shell
cat /proc/version
Linux version 5.4.0-144-generic (buildd@lcy02-amd64-069) (gcc version 7.5.0 (Ubuntu 7.5.0-3ubuntu1~18.04)) #161~18.04.1-Ubuntu SMP Fri Feb 10 15:55:22 UTC 2023
```

然后从github下载spdk，如果遇到了网络问题，可以手动修改github的dns映射，参考https://www.cnblogs.com/melodyjerry/p/13031571.html

```shell
# 下载
$ git clone https://github.com/spdk/spdk
$ cd spdk
# 安装子模块.
$ git submodule update --init
#如果安装的慢，可以将.gitmodules文件中的路径改成国内镜像的：hub.fastgit.org(只能用作镜像，不可访问)
# 安装依赖。有可能需要给pip换源，最好使用清华源
$ sudo ./scripts/pkgdep.sh
#编译
$ ./configure
$ make
```

安装依赖的时候报错

```
无法定位软件包 libfuse3-dev
```

安装的spdk版本出错，应该安装22.05

```shell
git clone --branch v22.05.x https://github.com/spdk/spdk.git
```

==应该在可以翻墙的Linux系统中安装。连子组件安装完之后一起上传到服务器。==



为了方便操作，我们使用vscode连接远程的服务器.注意，Host的命名不能加括号

```shell
Host 14#内网
    HostName 192.168.xxx.xxx
    User username
```



## SPDK结合FIO测试磁盘性能

安装完SPDK和fio之后，需要配置SPDK使用fio

```shell
./configure --with-fio=/home/fio --with-ocf
make
```

make之后，就可以看到在<spdk_repo>/build/fio目录会有下面两个文件，这就是fio_plugin的可执行程序。

要在 fio 中使用 SPDK fio 插件，请在运行 fio 时使用 LD_PRELOAD 指定插件二进制文件，并在 fio 配置文件中设置 ioengine=spdk（请参阅 spdk/spdk/tree/v22.05.x/examples/nvme/fio_plugin 目录下的 example_config.fio）

.fio文件内容

```shell
[global]
ioengine=spdk
thread=1
group_reporting=1
direct=1
verify=0
time_based=1
ramp_time=0
runtime=2
iodepth=128
rw=randrw
filename=/dev/nvme1n1p1
[test]
numjobs=1
```

### 

### 初始化NVMe SSD

使用lsblk的时候可以看出来对磁盘进行分区了，所以需要对该磁盘进行初始化

![image-20240321221737551](/images/搭建SPDK和fio环境/image-20240321221737551.png)

### 基于NVMe的fio_plugin

#### 把nvme SSD设备绑定到spdk

在运行SPDK应用程序之前，必须分配一些较大的页面，并且必须从本机内核驱动程序解绑定任何NVMe和I/OAT设备。SPDK包含一个脚本，可以在Linux上自动执行这个过程。这个脚本应该作为根运行。它只需要在系统上运行一次。

```shell
sudo scripts/setup.sh
```

![image-20240321155635222](/images/搭建SPDK和fio环境/image-20240321155635222.png)

但是发现我们的两个nvme设备都在使用，没有办法绑定

### 基于bdev的fio_plugin

具体的使用方式可以看github.com/spdk/spdk/tree/v22.05.x/examples/bdev/fio_plugin这里的readme

基于bdev的fio_plugin是将I/O在SPDK块设备bdev之上进行发送。而基于裸盘的fio_plugin，I/O是直接到裸盘上进行处理。两者最大的差别在于I/O是否经过bdev这一层。因此，基于bdev的fio_plugin能够很好的评估SPDK块设备层bdev的性能。其编译安装与裸盘的fio_plugin完全相同。

首先需要











