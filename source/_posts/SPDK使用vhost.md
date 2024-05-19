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

在服务器中安装虚拟机(没有图形界面安装没有图形界面的虚拟机)https://blog.csdn.net/qq_32305507/article/details/119976049 但是该教程不管用

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

使用cdrom(ubuntu的iso的镜像文件)安装虚拟机(一定要在图形界面中运行)

```shell
sudo ../QEMU/qemu-7.2.0/build/qemu-system-x86_64  -hda ubuntu-server1.qcow2 -cdrom ./ubuntu-18.04.6-live-server-amd64.iso -boot d -m 2048 -enable-kvm
```

然后直接就会跳出来QEMU的界面，然后先不设置网络，一切都点ok

安装完之后直接退出界面，然后使用如下命令直接可以启动。

```shell
sudo ../QEMU/qemu-7.2.0/build/qemu-system-x86_64 -hda test.qcow2 -boot c -m 1024 
```



### 给虚拟机配置网络(官网方法)

其中需要==设置网络参数==详见https://blog.csdn.net/v6543210/article/details/117668461

在使用`ip a`命令的时候发现有`virbr0`,直接使用NAT模式

直接使用virbr0可以看https://blog.csdn.net/liukuan73/article/details/49231785这篇文章。但是看不明白。

还是使用桥接模式，但是还是不太管用。为了能够正确使用，对每一种网络连接方法进行深入了解

[QEMU关于网络配置的文档](https://wiki.qemu.org/Documentation/Networking)。

QEMU的networking主要有两部分

- 提供给guest的虚拟网络设备，例如PCI网卡
- 与虚拟网卡交互的网络后端，例如把packets放到host的网络中

默认情况下QEMU会给guest创建一个SLiRP的用户网络后端和一个虚拟网络设备(an E1000 PCI card for most x86 PC guests)

#### Network Backends

创建一个网络后端

```
-netdev TYPE,id=NAME,...
```

id是virtual network device和network backend连接的选项。id用于区分后端。每个虚拟网络设备对应一个后端

多数情况下使用user模式就够用了，之后是使用tap模式

1. User mode stack(SLIRP)：

   如果不指定nic，那么默认的就是user模式。以下三个命令等效

   ```
   qemu -m 256 -hda disk.img &
   
   qemu -m 256 -hda disk.img -net nic -net user & #使用 -net user 必须同 -net nic配合
   
   qemu-system-i386 -m 256 -hda disk.img -netdev user,id=network0 -device e1000,netdev=network0,mac=52:54:00:12:34:56 &
   ```

   有以下几点限制

   - 开销很大，所以性能不好
   - `ICMP(ping)`无法使用
   - guest不能从主机或者外部网络直接访问

   User Networking是通过slirp实现的，它在QEMU中提供了一个完整的TCP/IP堆栈，并使用该堆栈实现一个虚拟的NAT网络。下图是一个默认的网络

   <img src="/images/SPDK使用vhost/image-20240504143305762.png" alt="image-20240504143305762" style="zoom:67%;" />

​	Gateway上的IP是guest和host交互的端口。例如，"ssh 10.0.2.2 "将从客户机 ssh 到主机。在QEMU中加入下面的命令会改变网络配置为192.168.76.0/24而不是默认的(10.0.2.0/24)并且会9开始分配guest的动态IP，而不是从15开始。

```shell
-netdev user,id=mynet0,net=192.168.76.0/24,dhcpstart=192.168.76.9
```

​	可以通过`restrict`选项将guest单独隔离开。

1. TAP

   Tap网络后端利用主机中的tap网络设备，它能够提供高性能并且能创建虚拟的多个拓扑。但是，他需要根据操作系统配置网络拓扑。命令如下

   ```shell
   -netdev tap,id=mynet0
   ```

#### Virtual Network Devices

**如何创建虚拟网络设备**

虚拟网络设备的选择需要基于需求和guest的环境。例如，如果要使用嵌入式网卡，需要使用`-nic`配置。

对于现在的guest，应该使用虚拟网络(半虚拟化)网络适配器，因为它具有最佳性能，但它需要特殊的guest驱动程序支持，这在非常旧的操作系统上可能无法获得。

使用`-device`可以添加一个虚拟的网络设备

```shell
-device TYPE,netdev=NAME
```

这里的`netdev`是前面的`id`。也可以使用`-nic`选项

```
-netdev user,id=n1 -device virtio-net-pci,netdev=n1
```

和

```
-nic user,model=virtio-net-pci
```

是等价的。

#### 如何配置

1. 如何给guest设置ssh权限

   最简单的命令如下，第一行创建虚拟的e1000网络设备，第二行创建了一个user类型的后端，转发local端口5555到guest的22。

   ```shell
   -device e1000,netdev=net0
   -netdev user,id=net0,hostfwd=tcp::5555-:22
   ```

   然后使用`ssh localhost -p 5555`就可以连接到虚拟机了

2. 在Linux中设置taps

   对于支持iproute2和tap/tun的Linux，可以通过如下配置。其中需要注意host物理机的配置，因为创建的网桥会成为物理设备的新端点，会导致host的网络中断

   ```
    # ip link add br0 type bridge
    # ip tuntap add dev tap0 mode tap
    # ip link set dev tap0 master br0   # set br0 as the target bridge for tap0
    # ip link set dev eth0 master br0   # set br0 as the target bridge for eth0
    # ip link set dev br0 up
   ```

   结构如下图所示

   ![image-20240505194322805](/images/SPDK使用vhost/image-20240505194322805.png)

   上面运行之后，网桥能够成功运行，但是不能成功使用，因为它没有IP address。用如下命令重新分配IP地址

   ```
    # ip address delete $PREFIX dev eth0  // 给eth0网卡删除对应的ip地址,$PREFIX应该是之前网卡的ip地址
    # ip address add $PREFIX dev br0    // 添加ip地址，该IP地址分配给网桥设备本身的，用于管理和通信。
    # ip route add default via $ROUTE dev br0  // 将默认路由添加到网桥 br0 上，并指定网关地址为 $ROUTE。网关是一个网络设备或者主机，用于连接两个不同网络，并且通常是两个不同网络之间的出口点。当你设置网关地址时，你在告诉主机或者网络设备，如果要与不在本地网络中的主机通信，应该将数据包发送到这个地址。网关地址通常是主机或者设备连接到的本地网络的路由器或者防火墙的接口地址。
   ```

   可以使用 shell 脚本自动在远程主机上设置 tap 网络，将物理设备的主控设置为网桥后，连接将丢失。

   但是按照上述的命令创建网桥之后host无法联网了。


### 在ubuntu18.04中使用netplan配置网桥，然后再配置tap提供给虚拟机

参考这篇文章：[Ubuntu网络配置-桥接和多网卡绑定](https://blog.csdn.net/shadow6907/article/details/138283306)。

我们的具体配置如下

```
# Let NetworkManager manage all devices on this system

network:
	version: 2
    ethernets:
    	eno1:
        	dhcp4: false
            dhcp6: false
    bridges:
    	br0:
        	interfaces: [eno1]
            dhcp4: false
            addresses: [192.168.1.14/24]
            routes:
            	- to: 0.0.0.0/0
                via: 192.168.1.1
            nameservers:
            	addresses: [114.114.114.114,8.8.8.8]
            parameters:
            	stp: false
            dhcp6: false
```

其中，eno1是之前网卡的名称，192.168.1.14是之前网卡的ip地址。

在给网桥配置路由的时候，to那里要改成0.0.0.0/0

之后使用命令更新网络

```shell
sudo netplan generate
sudo netplan --debug apply
```

之后仍可以使用192.168.1.14网址远程连接该机器。使用`ifconfig`查看网络设备

<img src="/images/SPDK使用vhost/image-20240507093011123.png" alt="image-20240507093011123" style="zoom:100%;" />

br0的ip已经变成了之前eno1的ip。通过下面的命令再创建一个TAP 设备，作为 QEMU 一端的接口：

```
tunctl -t tap0 -u root              # 创建一个 tap0 接口，只允许 root 用户访问
brctl addif br0 tap0                # 在虚拟网桥中增加一个 tap0 接口
ifconfig tap0 0.0.0.0 promisc up    # 启用 tap0 接口
```

删除`tap0`的命令

```
ip link set tap0 down
brctl delif br0 tap0
tunctl -d tap0
```

运行`brctl show`可以看到

![image-20240507103447017](/images/SPDK使用vhost/image-20240507103447017.png)

之后将上面命令转化为脚本见[文章](https://github.com/QthCN/opsguide_book/blob/master/QEMU%E7%BD%91%E7%BB%9C%E6%93%8D%E4%BD%9C%E7%9B%B8%E5%85%B3%E8%AF%B4%E6%98%8E%E5%8F%8A%E5%B8%B8%E7%94%A8%E5%91%BD%E4%BB%A4.md)，启动的时候使用。`qemu-ifup`

```shell
#!/bin/bash
switch=br0

if [ -n "$1" ]; then
#tunctl -u ${whoami} -t $1
ip link set $1 up
sleep 1
brctl addif ${switch} $1
exit 0
else
echo "Error: no interface specified"
exit 1
fi
```

并准备结束脚本

```shell
#!/bin/bash
switch=br0

if [ -n "$1" ];then
tunctl -d $1
brctl delif ${switch} $1
ip link set $1 down
exit 0
else
echo "Error: no interface specified"
exit 1
fi
```

之后通过如下命令启动虚拟机

```
qemu-system-x86_64 -hda test.qcow2 -smp 2 -m 1024 -net nic -net tap,ifname=tap1,script=/root/kvm_demo/qemu-ifup,downscript=/root/kvm_demo/qemu-ifdown -enable-kvm
```

在虚拟机中查看ip地址如下图所示

![image-20240507104901880](/images/SPDK使用vhost/image-20240507104901880.png)

成功联网。然后给虚拟机配置ssh操作之后就可以直连了

```
sudo apt update
sudo apt install openssh-server
sudo systemctl start ssh
systemctl enable ssh.service	#开机自启动
```

#### 使用netplan给虚拟机配置静态ip

使用命令`ip addr show`查看ip地址是动态的还是静态的。

如果网络接口已配置为静态IP地址，则在输出中会看到以下内容：

```
inet <静态IP地址>/<子网掩码> brd <广播地址> scope global <网络接口名称>
```

如果网络接口已配置为动态IP地址，则在输出中会看到以下内容：

```
inet <动态IP地址>/<子网掩码> brd <广播地址> scope global dynamic <网络接口名称>
```

如果是动态的，就修改  `/etc/netplan`中对应的文件。记得提前先把之前的备份了

![image-20240507171353225](/images/SPDK使用vhost/image-20240507171353225.png)

#### NAT网络和桥接模式的区别

NAT（Network Address Translation）和桥接（Bridge）是两种不同的网络连接方式。

==NAT==

 用于连接内部网络与外部网络，并隐藏内部网络的结构。在NAT模式下，虚拟机的网络适配器（Network Adapter）通过虚拟网络设备和虚拟化软件创建一个私有网络。虚拟机会被分配到私有网络中的一个IP地址，并使用该地址进行内部通信。

当虚拟机需要与外部网络通信时，虚拟化软件会将虚拟机的网络流量转发到宿主机（Host Machine）上。宿主机通过Network Address Translation（NAT）技术将虚拟机的私有IP地址转换成宿主机在外部网络上可路由的公共IP地址，同时维护一个转发表来跟踪虚拟机的网络连接。

==桥接==

将多个网络接口连接在一起形成一个单一的网络。虚拟机会和主机IP在同一个网段中。将虚拟机出来的计算机,直接连入当前的网络环境中,并且独占IP。**特点**：在当前网络中的全部计算机,都可以访问虚拟机。

<img src="/images/SPDK使用vhost/image-20240507100108525.png" alt="image-20240507100108525" style="zoom:80%;" />







### 问题注意

- 如果要使用user模式，QEMU7.2.0将slirp模块移除了，所以需要再安装回来


```
sudo apt-get install libslirp-dev-enable-kvm
```

然后在编译前

```
./configure --enable-slirp
```

- QEMU加载网卡显示failed with status 256

  脚本文件的权限问题，需要将脚本文件设置为777才可以

### 设置nographic启动

## QEMU配置SPDK

### 启动SPDK vhost target

先使用命令启动SPDK并绑定设备，给SPDK分配大页

```shell
sudo HUGEMEM=4096 scripts/setup.sh
```

如果要解绑，`sudo scripts/setup.sh reset`

然后启动SPDK vhost target应用。下面的命令会在CPU核0和1上启动vhost，所有未来的套接字文件都放在/var/tmp中。Vhost将完全占用给定的CPU核进行I/O轮询。vhost设备可以被限制在这些CPU内核的子集上运行。

```shell
sudo build/bin/vhost -S /var/tmp -m 0x3
```



###  创建bdev(block device)

SPDK bdevs是提供给Guest的块设备。**vhost-scsi**中，bdev在客户端中作为SCSI lun暴露在连接到vhost-scsi控制器的SCSI设备上。**vhost-blk**中，bdevs直接作为客户端的块设备。

（注意：SPDK bdev是SPDK中对多种存储后端(storage backend)的抽象。 这些存储后端(storage backend)包括：ceph RBD，ramdisk，NVMe，iSCSI，逻辑卷，甚至是virtio）。这里就体现了SPDK block device layer的概念。可以在 [Block Device User Guide](https://spdk.io/doc/bdev.html) 找到更多的关于SPDK存储后端的信息

bdev设备我们直接选择使用NVMe bdev

```
sudo ./scripts/setup.sh status   #查找nvme设备对应的PCI地址
rpc.py bdev_nvme_attach_controller -b NVMe1 -t PCIe -a 0000:01:00.0  #在SPDK中创建NVMe Bdev
```

输出结果

![image-20240511205528041](/images/SPDK使用vhost/image-20240511205528041.png)

<u>试一下使用 bdev_split_create 能不能成</u>

### 创建vhost device

选择试用vhost-blk作为客户端的块设备。下面的命令使用NVMe1n1创建vhost-blk设备。该设备会QEMU可以通过`/var/tmp/vhost.1`访问该设备。所有的I/O轮询都将被固定到给定的cpumask中占用最少的CPU内核上——在本例中是CPU 0。对于NUMA系统，cpumask应该在与其关联的VM相同的CPU套接字上指定内核。

```
 sudo ./rpc.py vhost_create_blk_controller --cpumask 0x1 vhost.1 NVMe1n1
```

为了方便之后的使用，将当前的rpc的配置生成文件

```
./rpc.py save_config > vhost.json
```

之后可以直接在配置文件中直接加载vhost

```
build/bin/vhost -S /var/tmp -m 0x3 -s 1024 -c vhost.json
```



### 通过QEMU启动

在QEMU启动的参数中加入必要的参数才可以将虚拟机连接到vhost

第一，给虚拟机指定内存，由于QEMU必须与SPDK vhost target共享虚拟机的内存，因此需要设置`share=on`

```
-object memory-backend-file,id=mem,size=1G,mem-path=/dev/hugepages,share=on
-numa node,memdev=mem
```

这里容易出错

其次，确定QEMU从虚拟机的映像中boots，需要设置`bootindex=0`

```
-drive file=guest_os_image.qcow2,if=none,id=disk
-device ide-hd,drive=disk,bootindex=0
```

如下参数指定SPDK vhost设备

```
-chardev socket,id=char1,path=/var/tmp/vhost.1
-device vhost-user-blk-pci,id=blk0,chardev=char1
```

QEMU启动的命令

```shell
qemu-system-x86_64 -smp 2 -m 1G\
-net nic -net tap,ifname=tap1,script=qemu-ifup,downscript=qemu-ifdown \
-enable-kvm \
-drive file=test.qcow2,if=none,id=disk \
-device ide-hd,drive=disk,bootindex=0 \
-object memory-backend-file,id=mem,size=1G,mem-path=/dev/hugepages,share=on \
-numa node,memdev=mem \
-chardev socket,id=char1,path=/var/tmp/vhost.1 \
-device vhost-user-blk-pci,chardev=char1,num-queues=2 
```

如果遇到了报错：`qemu total memory for NUMA nodes should equal RAM size`是因为 `-m`指定的值和共享内存的`size`大小不一样

运行上述命令出现如下错误

![image-20240511221538534](/images/SPDK使用vhost/image-20240511221538534.png)

这种问题一般是spdk的运行中出现了问题。可以重新make一下spdk，看一下SPDK的代码中是否有问题。

进入虚拟机后执行命令`lsblk --output "NAME,KNAME,MODEL,HCTL,SIZE,VENDOR,SUBSYSTEMS"`可以看到出现虚拟设备vda. vda是SPDK的vhost-blk disk

![image-20240516134856410](/images/SPDK使用vhost/image-20240516134856410.png)

## 多个QEMU虚拟机同时访问一块物理盘

### 虚拟机克隆

只是使用qemu的话克隆虚拟机比较麻烦，所以还是一个个创建吧。



### 多个虚拟机能不能设置同一个vhost device

可以，创建多个虚拟机，然后可以使用同一个device

## 参考文献

https://rootw.github.io/2018/05/SPDK-all/   

[浅析SPDK技术：vhost](https://blog.csdn.net/anyegongjuezjd/article/details/136123555)
