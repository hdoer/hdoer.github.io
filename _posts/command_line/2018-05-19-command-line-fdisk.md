---
title: "command line - fdisk"
# subtitle: "fdisk 命令"
layout: post
author: "Wentao Dong"
date: 2022-06-20 21:00:00
catalog: false
header-style: post
header-img: "img/city_night.png"
tags:
  - Command
  - Linux

---

使用桌面Windows/Linux系统用U盘拷贝数据时，直接插到USB接口上，过一会儿就能在文件管理器里看到挂载的U盘了。但是如果装的是Ubuntu Server系统，就没这么方便了。下面简单介绍一下fdisk 和 mount命令

#### 操作系统

```
lsb_release -a

output:
Distributor ID:    Ubuntu
Description:    Ubuntu 20.04.4 LTS
Release:    20.04
Codename:    focal
```

#### 查看Linux硬盘信息

```
sudo fdisk -l
```

#### 更改分区表

```
sudo fdisk $partition
```

#### 格式化分区

```
sudo mkfs.ext4 $partition
```

注: 分区不要选错了

#### 创建挂载目录

```
sudo mkdir $dir
```

#### 挂/卸载磁盘

```
sudo mount $partition $dir
sudo umount $dir
```

#### 查看磁盘分区UUID

```
sudo blkid
```

#### 配置开机自动挂载

```
sudo vim /etc/fstab
```

参考/etc/fstab文件中记录和blkid输出增加一行，$partition-uuid即为blkid输出的UUID

```
/dev/disk/by-uuid/$partition-uuid /bigdata ext4 defaults 0 2
```

/etc/fstab 字段解释

```
<file system>  <mount point>  <type>  <options>  <dump>  <pass>
```

1. file system: 分区信息，有多种可配 /dev/disk/by-uuid｜/dev/disk/by-partuuid 等，具体查看/dev/disk 文件

2. mount point: 挂载目录

3. type : 分区格式化类型，上面用mkfs.ext4创建的为ext4

4. options: 默认设置defaults，等于rw,suid,dev,exec,auto,nouser,async，具体含义：
   
   1. 自动/手动挂载:auto/noauto
   
   2. 读写权限:ro(只读)/rw(读写)
   
   3. 可执行:exec(分区中允许二进制文件执行)/noexec(不允许执行)
   
   4. I/O同步:sync(I/O以同步方式进行)/async(非同步)
   
   5. 户挂载权限:user(允许任何用户挂载设备)/nouser(仅root用户挂载)
   
   6. 临时文件执行权限：suid(允许对 suid 和 sgid 位进行操作。它们主要用于允许计算机系统上的用户以临时提升的权限执行二进制可执行文件，以执行特定任务。)/nosuid(不允许) 

5. dump: 是否使用dump对该文件系统备份，0(忽略)/1(备份)，大部分用户没有安装dump，设置为0即可

6. pass: 是否使用fsck对该文件系统执行扇区检验，0(不检验)/1(最早检验)/2(在1之后检验)，一般根目录配置1, 其他配置2即可

挂载/etc/fstab配置的全部文件系统

```
sudo mount -a
```
