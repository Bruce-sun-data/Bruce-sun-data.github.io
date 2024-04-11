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

要在 fio 中使用 SPDK fio 插件，请在运行 fio 时使用 LD_PRELOAD 指定插件二进制文件。

```shell
sudo LD_PRELOAD=<path to spdk>spdk/build/fio/spdk_bdev ./fio/fio ./test_files/example_spdk_bdev.fio
```

SPDK提供两种形态的fio_plugin：

- 基于裸盘NVMe的0fio_plugin，其特点为I/O通过SPDK用户态驱动直接访问裸盘，常用于评估SPDK用户态驱动在裸盘上的性能。
- 基于bdev的fio_plugin，其特点为I/O测试基于SPDK块设备bdev之上，所有I/O经由块设备层bdev，再传送至裸盘设备。常用于评估SPDK块设备bdev的性能。

### 初始化NVMe SSD

在运行SPDK应用程序之前，必须分配一些较大的页面，并且必须从本机内核驱动程序解绑定任何NVMe和I/OAT设备。SPDK包含一个脚本，可以在Linux上自动执行这个过程。这个脚本应该作为根运行。它只需要在系统上运行一次。

```shell
sudo scripts/setup.sh
```

![image-20240321155635222](/images/搭建SPDK和fio环境/image-20240321155635222.png)

但是发现我们的两个nvme设备都在使用，没有办法绑定。并且使用lsblk的时候可以看出来对磁盘进行分区了，所以需要对该磁盘进行初始化

![image-20240321221737551](/images/搭建SPDK和fio环境/image-20240321221737551.png)

先使用fdisk删除分区。然后使用nvme-cli相关命令初始化nvme ssd

```shell
sudo nvme format /dev/nvme1n1
```

再运行上面的绑定命令，出现类似结果则绑定成功。再使用`lsblk`就不会看到结果了

![image-20240322093108395](/images/搭建SPDK和fio环境/image-20240322093108395.png)

绑定之后，在spdk的路径运行

```shell
sudo ./build/examples/hello_world
```

结果如下

![image-20240322093927566](/images/搭建SPDK和fio环境/image-20240322093927566.png)

成功

### 基于NVMe的fio_plugin

在 fio 配置文件中设置 ioengine=spdk（请参阅 github.com/spdk/spdk/tree/v22.05.x/examples/nvme/fio_plugin 目录下的 example_config.fio）

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
size=16384
rw=randrw

[test]
numjobs=1
filename=trtype=PCIe traddr=0000.05.00.0 ns=1
```

运行命令如下

```shell
LD_PRELOAD=<path to spdk>spdk/build/fio/spdk_nvme fio example_spdk_nvme.fio
```

运行结果如下

![image-20240322102259191](/images/搭建SPDK和fio环境/image-20240322102259191.png)

### 基于bdev的fio_plugin

具体的使用方式可以看github.com/spdk/spdk/tree/v22.05.x/examples/bdev/fio_plugin这里的readme

基于bdev的fio_plugin是将I/O在SPDK块设备bdev之上进行发送。而基于裸盘的fio_plugin，I/O是直接到裸盘上进行处理。两者最大的差别在于I/O是否经过bdev这一层。因此，基于bdev的fio_plugin能够很好的评估SPDK块设备层bdev的性能。其编译安装与裸盘的fio_plugin完全相同。也需要将盘从原始驱动中解除。

使用下面的命令自动监测，生成bdev.json

```shell
scripts/gen_nvme.sh --json-with-subsystems > /tmp/bdev.json
```

如下所示

```json
{
    "subsystems": [
        {
            "subsystem": "bdev",
            "config": [
                {
                "method": "bdev_nvme_attach_controller",
                "params": {
                    "trtype": "PCIe",
                    "name":"Nvme1",
                    "traddr":"0000:05:00.0"
                    }
                }
            ]
        }
    ]
}
```

.fio文件内容

```shell
[global]
ioengine=spdk_bdev
spdk_json_conf=/tmp/bdev.json
thread=1
group_reporting=1
direct=1
verify=0
time_based=1
ramp_time=0
runtime=10
iodepth=64
size=16384
rw=randwrite

[test]
bs=16k
filename=Nvme1n1
```

**注意 Nvme0是指controller，而Nvme0n1才是指bdev，在Linux kernel中亦是如此，而非SPDK限定**

运行命令如下

```shell
LD_PRELOAD=<path to spdk>spdk/build/fio/spdk_bdev fio example_spdk_bdev.fio
```

结果如下

<img src="/images/搭建SPDK和fio环境/image-20240322150518070.png" alt="image-20240322150518070"  />



### 使用fio运行自己定义的工作负载

参考链接https://fio.readthedocs.io/en/latest/fio_doc.html#cmdoption-arg-read_iolog

==**write_iolog**=str==可以将fio中执行一次的流量写入到对应的文件中

==**read_iolog**=str==可以去读对应的流量文件然后执行。注意，一定要把时间线往后拉，这样才能都执行完

这两个参数都可以在.fio配置文件中直接实现

```
fio version 3 iolog
2 Nvme1n1 add
872 Nvme1n1 open
878 Nvme1n1 write 0 16384
888 Nvme1n1 write 0 16384
891 Nvme1n1 write 0 16384
10001677 Nvme1n1 close
```

这是流文件格式，分别对应 `timestamp(μs) filename action offset length(Byte)`

### 收集fio执行的每段结果并输出

在命令行中使用三个参数

```shell
-output=../test_files/output #指定输出的具体文件
-output-format=json #指定输出的格式
-status-interval=1 #指定输出的时间间隔
```

但是我们使用fio官方文档中的 ==Measurements and reporting==模块进行信息收集

在.fio配置文件中加入命令

```shell
write_bw_log=./test_files/foo
write_lat_log=./test_files/foo
log_avg_msec=10
```

每隔10ms收集一次数据

为了比较使用trace文件方式和fio本身生成流量方式的区别。首先设置.fio配置文件为

```shell
[global]
ioengine=spdk_bdev
spdk_json_conf=/home/dell6/szb/NVMe/fio/test_files/bdev.json
thread=1
group_reporting=1
direct=1
verify=0
time_based=1
ramp_time=0
runtime=5
iodepth=64
size=16384
rw=randwrite
write_bw_log=./test_files/foo
write_lat_log=./test_files/foo
log_avg_msec=10
write_iolog=./test_files/iolog1.txt

[test]
bs=16k
filename=Nvme1n1
```

运行结果如下图

![image-20240323185108469](/images/搭建SPDK和fio环境/image-20240323185108469.png)

同时还会生成相关的统计数据文件

![image-20240323183452664](/images/搭建SPDK和fio环境/image-20240323183452664.png)

每个值的意义见链接https://fio.readthedocs.io/en/latest/fio_doc.html#log-file-formats

然后将write改成read再试一次

![image-20240323185130498](/images/搭建SPDK和fio环境/image-20240323185130498.png)

结果近似，并且多次只运行trace文件也是有误差的。



## SPDK日志输出





# 实验过程

## 多租户相同发送速率竞争同一块磁盘

使用基于bdev的fio_plugin测试。因为我们修改的其实是bdev块设备的调度策略。我们使用fio运行我们自己负载。

我们自己生成流量数据，然后在.fio中使用参数`read_iolog=./test_files/iolog.trace`可以读取自己的流量数据。通过设置`runtime`的值可以防止段错误，即还有命令在跑但是程序结束的问题

### 16KB的数据

![image-20240324211128634](/images/搭建SPDK和fio环境/image-20240324211128634.png)

差距有点大，再试一下4KB的数据。

### 4KB的数据

设置iosize=4KB，基准测试中配置如下,iops的值是79.5K。为了让3个流都能被完全服务，所以在生成流量的时候，iosize设置为4KB，时间间隔是1000/26.5=38μs

```shell
[global]
ioengine=spdk_bdev
spdk_json_conf=/home/dell6/szb/NVMe/fio/test_files/bdev.json
thread=1
group_reporting=0
direct=1
verify=0
time_based=1
ramp_time=0
runtime=10
iodepth=1
bs=4k
size=4k
rw=randwrite
[test1]
numjobs=1
filename=Nvme1n1
```

通过设置多个job区域，实现多线程竞争的模式

![image-20240325105430535](/images/搭建SPDK和fio环境/image-20240325105430535.png)

![image-20240325105457953](/images/搭建SPDK和fio环境/image-20240325105457953.png)

## 多租户不同发送速率竞争同一块磁盘

### 4KB数据

经过上面的判断，设置三个任务，其中一个任务在150000条之后把时间间隔改为20μs，提高发送速率

![image-20240325152824718](/images/搭建SPDK和fio环境/image-20240325152824718.png)

![image-20240325153007846](/images/搭建SPDK和fio环境/image-20240325153007846.png)

## 在使用fio的路径中修改SPDK源码，实现简单的漏桶控制

问题：是把算法实现在`lib/bdev`还是实现在`lib/nvme`?

![image-20240328210403556](/images/搭建SPDK和fio环境/image-20240328210403556.png)



### SPDK QoS机制

SPDK有自己的QoS实现机制。但是spdk qos做在bdev层，是面向块设备的qos，不是面向客户端的qos。

可以自己生成多个虚拟的块设备，然后再使用QoS机制。

### 源码分析

==从fio开始分析,bdev==

关于spdk在bdev层的流程可以看https://rootw.github.io/2018/05/SPDK-ioanalyze/

在`./spdk/examples/bdev/fio_plugin/fio_plugin.c`中找到每个租户(thread)创建的过程，

`spdk_fio_init_thread()`函数创建spdk_thread，创建的线程数比作业数目多1。调用的是`./spdk/lib/thread/thread.c`中的`spdk_thread_create()`函数，创建thread。为了调试，每次修改，应该同时对`fio_plugin`和`thread`进行重make。`spdk_fio_bdev_open`调用了作业数目的次数。

`./spdk/lib/bdev` 每调用一次`spdk_fio_bdev_open()`都会调用`bdev_open()`.



通过`bdev_channel_get_io()`获取`bdev_io`，然后调用`bdev_io_submit()`，该函数的调用次数比请求的总次数多三次。但是因为我们的请求都是写，所以`bdev_write_blocks_with_md()`该函数调用的次数和请求的数目是一致的。

io channel是一个线程强相关对象，不同的线程对应不同的channel，。request必须在关联的I/Ochannel上提交。

`fio_plugin.c`中的`spdk_fio_queue()`用于提交IO request









==根据NVMe执行过程分析==

在`nvme_ctrlr.c`中找到`nvme_ctrlr_proc_add_io_qpair()`，被调用的次数和线程数目一致

`nvme_qpair_submit_request()`负责从把request放到qpair中。



### 在bdev层进行token对每个thread进行速率限制

首先在`./spdk/lib/bdev`中创建 `resource_schedule.c`文件,该文件中完成速率控制的函数。然后在。`./include/spdk/bdev.h`中创建`shceduler`结构体。

该结构体中包括请求排队队列，和一些相关的参数。并且C语言中没有实现map。

给`spdk_thread`中加入 void*指针，指向scheduler，每个线程绑定一个scheduler，每个线程创建之后将scheduler绑定到该线程

在`_bdev_io_submit()`中，在记录

**设置三个任务，其中一个任务在150000条之后把时间间隔改为20μs，提高发送速率**













