---
title: PVE+RANCHER：一种低成本的云集群方案
date: 2024-12-08 21:57
tags:
---

本文主要介绍我实验室当前使用的云服务集群，当前架构版号V24.1.28_1,以下架构均已通过生产环境验证并稳定运行至少6个月。

我们希望集群能够承担以下负载能力:
1. 数据可靠，具有去单点故障能力、热备能力
2. 弹性化扩缩容
3. 完成包括GPU调度等问题

# 架构介绍

以下我按照IAAS，PAAS，SAAS层次介绍

## IAAS层次

### 虚拟化方案：proxmox ve（KVM）

我们建议对于刚上手虚拟化的人，选择成熟可靠好上手的架构，不要选择openstack这样的大架构，根本玩不起来,我并不想介绍为什么不推荐ESXI，那样会带来争议，
我选择PVE单纯因为开源免费公开透明。

### 数据方案：

1. 本地存储：ZFS

**注**：要会用vdev，在进行容量更换，raid等级更换，用vdev非常方便，我们曾经把根系统变到64GU盘，又折腾到硬盘，然后又从500G raid1 换到300G raid1，完全不需要重装。

**关于zfs改内存**
* 如果区域没有读性能需求，比如系统区域，我直接给512MB，对于备份区域，可以直接关闭了ARC
* 如果区域有比较高的读性能需求，可以按照PVE默认来分配（内存的1/8），ARC对读取的提升是显而易见的



***以下是IO测试数据，在一个双机械盘的ZFS上，机械盘为SAS 600G 15K***

命令如下

```
1. BW-R
fio -name=Seq_Read_IOPS_Test -group_reporting -direct=1 -iodepth=128 -rw=read -ioengine=libaio -refill_buffers -norandommap -randrepeat=0 -bs=4k -size=5G -numjobs=1 -runtime=600 -filename=./test
2.BW-W
fio -name=Seq_Write_IOPS_Test -group_reporting  -direct=1 -iodepth=128 -rw=write -ioengine=libaio -refill_buffers -norandommap -randrepeat=0 -bs=4k -size=5G -numjobs=1 -runtime=600 -filename=./test
3.IOPS-R
fio -name=Rand_Read_IOPS_Test -group_reporting -direct=1 -iodepth=128 -rw=randread -ioengine=libaio -refill_buffers -norandommap -randrepeat=0 -bs=4k -size=1G -numjobs=1 -runtime=600 -filename=./test
4.IOPS-W
fio -name=Rand_Write_IOPS_Test -group_reporting -direct=1 -iodepth=128 -rw=randwrite -ioengine=libaio -refill_buffers -norandommap -randrepeat=0 -bs=4k -size=1G -numjobs=1 -runtime=600 -filename=./test
5.IOPS-Hybid
fio -name=Read_Write_IOPS_Test -group_reporting -direct=1 -iodepth=128 -rw=randrw -rwmixread=70 -refill_buffers -norandommap -randrepeat=0 -ioengine=libaio -bs=4k -size=1G -numjobs=1 -runtime=600 -ioscheduler=noop -filename=./test
6.Throughput-R
fio -name=Read_BandWidth_Test -group_reporting -direct=1 -iodepth=32 -rw=read -ioengine=libaio -refill_buffers -norandommap -randrepeat=0 -bs=1024k -size=5G -numjobs=1 -runtime=600 -filename=./test
7.Throughput-W
fio -name=Write_BandWidth_Test -group_reporting -direct=1 -iodepth=32 -rw=write -ioengine=libaio -refill_buffers -norandommap -randrepeat=0 -bs=1024k -size=5G -numjobs=1 -runtime=600 -filename=./test
```

结果如下

```
系统存储区域(local-zfs)
带宽
顺序读:679MB/S
顺序写:53MB/S
IOPS
随机读:24K
随机写:483
混合读写:读 1070 写 457
吞吐量
读:1984MB/S
写:50.5MB/S
```

2. 共享存储：CEPH

**注**：
1. 如果真的准备用ceph，必须要有万兆网络

2. PVE8装ceph比较方便，如果是PVE<=7,得用反向代理，因为默认是从download.proxmox下，你反向代理到中科大的Proxmox源。（你搜一下，用nginx就行）

3. 如果需要ceph的高级功能（如RGW等），必须把ceph和PVE脱钩，也就是使用cephadm安装ceph。记得给podman配好代理，要不然拉不下来镜像。

**CEPH配置建议**

1. 建议配置DB

注意ceph写入流程：客户端向主PG写，主PG向副本写，只要WAL写结束，写入就会ACK。

所以这就是为什么“DB要比block区域快，WAL区域比DB快”，日志设备快，写入完成就快，写IO延迟就小。但是实际上你不需要单独配置WAL，它会使用DB区域

DB大小：RBD需求就是block区域大小的1%-2%，对象存储RGW，就要4%。

我建议配置DB，因为我们之前写入延迟10%，配置以后，稳定降低到0.5%。

2. 设备分类

我很喜欢CEPH的一个点就在于配合device class和crush rule可以让数据选择性落入我希望的设备中。

因为现实中包括我们实验室存储设备包括4t 7.2k hdd，300g 10k hdd，nvme等设备。

我们肯定希望对象存储扔到大容量机械盘，rbd放10k的高速盘，元数据放nvme。

所以简单方案就是，新建一个class叫hddl，然后新建crush rule，把rgw.data等不需要低延迟的数据区改成这个rule就行。

3. 怎么将pve和cephadm配合使用

pve添加ceph存储即可

4. 故障域

建议放host


