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

应该在可以翻墙的Linux系统中安装。连子组件安装完之后一起上传到服务器。



为了方便操作，我们使用vscode连接远程的服务器.注意，Host的命名不能加括号

```shell
Host 14#内网
    HostName 192.168.xxx.xxx
    User username
```













