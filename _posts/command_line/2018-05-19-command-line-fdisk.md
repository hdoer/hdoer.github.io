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
   
   7. dev/nodev: 在挂载时，是否允许系统把该文件系统里面的 block/character 文件当作是 block/character 文件来处理。进一步解释见下文[fstab 中nodev为什么重要](#fstab 中nodev为什么重要)

5. dump: 是否使用dump对该文件系统备份，0(忽略)/1(备份)，大部分用户没有安装dump，设置为0即可

6. pass: 是否使用fsck对该文件系统执行扇区检验，0(不检验)/1(最早检验)/2(在1之后检验)，一般根目录配置1, 其他配置2即可

挂载/etc/fstab配置的全部文件系统

```
sudo mount -a
```

#### fstab 中nodev为什么重要

dev/nodev 选项意思是在挂载时，是否允许系统把该文件系统里面的 block/character 文件当作是 block/character 文件来处理

默认情况下对底层设备的访问只受文件权限控制，假如一个系统中有个设备/dev/sda，这个设备只允许root用户或root组中的用户访问。这个设备可能是个磁盘、摄像头、话筒。你是一个guest用户，且不在root组中，因此你没有权限访问该设备。

假设你想访问 /dev/sda （然后你几乎可以对磁盘的内容做任何你想做的事情，包括植入一个程序，让你成为 root）。如何做呢，我们举个例子就清楚了

先看下你想访问的目标系统中的设备信息：ls -l /dev/sda

```
brw-rw----  1 root disk      8,   0 Sep  8 11:25 sda
```

这是个块设备文件，主设备号为8，次设备号为0，只允许root用户或disk组中的用户访问。很可惜gust不在disk组，不过你可以使用nodev选项挂载USB设备

在另一台电脑上，你是root用户，你可以在你的USB设备上创建一个特殊的文件：

```
mknod -m 666 usersda b 8 0
```

没错这个特殊文件的主设备号和次设备号与你/dev/sda相同。而且你对这个特殊文件拥有读写权限

接下来你把这个USB设备挂载到目标系统中，然后你就可以通过这个usersda不受限制的访问之前不能访问的/dev/sda设备了

原因是内核使用这些主设备号和次设备号将设备专用文件与驱动程序进行匹配，因此可以拥有多个指向相同内核驱动程序和设备的文件。



参考链接

1. [why-is-nodev-in-etc-fstab-so-important](https://unix.stackexchange.com/questions/188601/why-is-nodev-in-etc-fstab-so-important-how-can-character-devices-be-used-for)
