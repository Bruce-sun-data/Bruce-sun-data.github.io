---
title: SPDK性能保障实验
date: 2024-06-23 16:16:04
published: false
tags:
typora-root-url: ..
categories:
    - 存储研究
---

# SPDK性能保障实验

## SPDK实验环境启动即部分配置

### 配置虚拟磁盘提供给虚拟机

启动SPDK

```shell
#绑定NVMe SSD,并给SPDK分配8GB的大页内存
sudo HUGEMEM=8192 scripts/setup.sh
#查看SPDK已经绑定的NVMe设备
sudo ./scripts/setup.sh status
#解绑SPDK
sudo scripts/setup.sh reset
```

![image-20240623170530256](/images/SPDK性能保障实验/image-20240623170530256.png)

可以看到和SPDK绑定的NVMe SSD设备的PCIe地址是0000:05:00.0

之后在`script`目录中利用`vhost.json`配置文件将NVMe1设置为虚拟磁盘

`vhost.json`的文件内容如下。里面的内容含义可以在官网提供的[JSON-RPC](https://spdk.io/doc/jsonrpc.html)中找到

```json
{
  "subsystems": [
    {
      "subsystem": "scheduler",
      "config": [
        {
          "method": "framework_set_scheduler",
          "params": {
            "name": "static",
            "period": 1000000
          }
        }
      ]
    },
    {
      "subsystem": "accel",
      "config": []
    },
    {
      "subsystem": "vmd",
      "config": []
    },
    {
      "subsystem": "sock",
      "config": [
        {
          "method": "sock_impl_set_options",
          "params": {
            "impl_name": "posix",
            "recv_buf_size": 2097152,
            "send_buf_size": 2097152,
            "enable_recv_pipe": true,
            "enable_quickack": false,
            "enable_placement_id": 0,
            "enable_zerocopy_send_server": true,
            "enable_zerocopy_send_client": false,
            "zerocopy_threshold": 0
          }
        }
      ]
    },
    {
      "subsystem": "bdev",
      "config": [
        {
          "method": "bdev_set_options",
          "params": {
            "bdev_io_pool_size": 65535,
            "bdev_io_cache_size": 256,
            "bdev_auto_examine": true
          }
        },
        {
          "method": "bdev_nvme_set_options",
          "params": {
            "action_on_timeout": "none",
            "timeout_us": 0,
            "timeout_admin_us": 0,
            "keep_alive_timeout_ms": 10000,
            "transport_retry_count": 4,
            "arbitration_burst": 0,
            "low_priority_weight": 0,
            "medium_priority_weight": 0,
            "high_priority_weight": 0,
            "nvme_adminq_poll_period_us": 10000,
            "nvme_ioq_poll_period_us": 0,
            "io_queue_requests": 512,
            "delay_cmd_submit": true,
            "bdev_retry_count": 3,
            "transport_ack_timeout": 0,
            "ctrlr_loss_timeout_sec": 0,
            "reconnect_delay_sec": 0,
            "fast_io_fail_timeout_sec": 0
          }
        },
        {
          "method": "bdev_nvme_attach_controller",
          "params": {
            "name": "NVMe1",
            "trtype": "PCIe",
            "traddr": "0000:05:00.0",
            "prchk_reftag": false,
            "prchk_guard": false,
            "ctrlr_loss_timeout_sec": 0,
            "reconnect_delay_sec": 0,
            "fast_io_fail_timeout_sec": 0
          }
        },
        {
          "method": "bdev_nvme_set_hotplug",
          "params": {
            "period_us": 100000,
            "enable": false
          }
        },
        {
          "method": "bdev_wait_for_examine"
        }
      ]
    },
    {
      "subsystem": "vhost_blk",
      "config": [
        {
          "method": "vhost_create_blk_controller",
          "params": {
            "ctrlr": "vhost.1",
            "dev_name": "NVMe1n1",
            "cpumask": "1",
            "readonly": false,
            "transport": "vhost_user_blk"
          }
        }
      ]
    },
    {
      "subsystem": "scsi",
      "config": null
    },
    {
      "subsystem": "nbd",
      "config": []
    },
    {
      "subsystem": "vhost_scsi",
      "config": []
    }
  ]
}
```

通过下面的命令创建vhostDisk。`vhost`命令可以使用`-h`查看每个参数的含义

```
sudo build/bin/vhost -S /var/tmp -m 0x3 -s 1024 -c ./scripts/vhost.json
```

![image-20240623173523597](/images/SPDK性能保障实验/image-20240623173523597.png)

### 启动两个虚拟机

使用向日葵远程启动这两个虚拟机

```shell
sudo sh ./startqemu.sh tap0 test.qcow2
sudo sh ./startqemu.sh tap1 test2.qcow2
```

此时的SPDK服务端也会有日志输出。`INFO`类型的日志

### 两个虚拟机使用FIO访问该虚拟磁盘

在主机中使用`pssh`命令可以同时在多个虚拟机中运行命令。

先在两个虚拟机的主目录中创建`script.sh`脚本文件，

```shell
#!/bin/bash
cat /dev/null >./result.txt
sudo ./fio/fio/fio ./testfile/libaio.fio >> ./result.txt
```

然后在server中创建`host.txt`记录每个虚拟机的ip

```
ubuntu@192.168.1.175
ubuntu@192.168.1.174
```

然后运行命令` pssh -h host.txt -l ubuntu -i "at 21:24 -f ./script.sh "`就可以看到每个虚拟机中的==./result.txt==中有相同内容。其中时间更改为自己想要的时间。

`at`是ubuntu的一个命令。如果时间输入出错，可以通过输入`atq`命令来查看所有的`at`任务列表。然后，可以使用`atrm`命令和你的任务编号来取消任务。例如，如果你的任务编号是5，你可以输入`atrm 5`来取消这个任务。

![image-20240623184419675](/images/SPDK性能保障实验/image-20240623184419675.png)

在两个虚拟机中都有结果保留



## 修改SPDK源码

### 日志系统与编译

使用`SPDK_NOTICELOG`函数可以输出对应内容

首先调试了==bdev.c==中的`bdev_io_submit`函数，观察是否有输出，即IO请求是否会经过`bdev`层

vhost_blk加入`SPDK_NOTICELOG`函数，查看输出

修改日志级别显示的代码

```shell
# 修改日志输出级别为DEBUG
sudo ./scripts/rpc.py log_set_print_level DEBUG
# 查看日志输出级别
sudo ./scripts/rpc.py log_get_print_level
# 设置日志级别
sudo ./scripts/rpc.py -s /var/tmp/spdk.sock log_set_level debug
# 查看日志级别
sudo ./scripts/rpc.py -s /var/tmp/spdk.sock log_get_level
# 设置日志标志
sudo scripts/rpc.py -s /var/tmp/spdk.sock log_set_flag vhost_blk
```

这里日志的flag是指DEBUG输出中的第一个参数。

![image-20240624211504045](/images/SPDK性能保障实验/image-20240624211504045.png)



修改之后在==./spdk/lib==目录下使用`make`重新编译。但是不管用，需要先关闭虚拟机然后停止虚拟磁盘然后再重新编译才管用。

### 查找代码部署的位置

对使用vhost的IO请求进行追踪，参考这篇[文章](https://rootw.github.io/2018/05/SPDK-ioanalyze/)。

==Gimbal==是将算法实现在了目录==./lib/nvmf/==中，后面将请求提交到了==Bdev==层

所以我们也可以将算法实现在目录==./lib/vhost/==中

即我们和Gimbal一样都将代码实现在SPDK的存储协议层













