---
title: "command line - adduser"
# subtitle: "adduser 命令"
layout: post
author: "Wentao Dong"
date: 2022-06-20 21:00:00
catalog: false
header-style: post
header-img: "img/city_night.png"
tags:
  - Command
  - Linux
  - Ubuntu
---

与服务器打交道，偶尔也需要创建/删除用户，尤其是多人共用一台开发机的情况，下面简单介绍几个用户管理方面的命令

#### 操作系统

```
lsb_release -a

output:
Distributor ID:    Ubuntu
Description:    Ubuntu 20.04.4 LTS
Release:    20.04
Codename:    focal
```

#### 增加/删除用户组

```
sudo addgroup [--gid ID] $GROUP
sudo delgroup $GROUP
```

#### 增加/删除用户

```
sudo adduser [--ingroup GROUP | --gid ID] $USER
sudo deluser $USER
```

#### 将用户加入/移除组

```
sudo adduser $USER $GROUP
sudo deluser $USER $GROUP
```

#### 修改用户密码

```
sudo passwd $USER
```
