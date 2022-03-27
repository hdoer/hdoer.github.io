---
title: "frame - thrift(一)"
subtitle: "thrift初印象"
layout: post
author: "Wentao Dong"
date: 2021-06-12 21:00:00
catalog: false
header-style: post
header-img: "img/city_night.png"
mermaid: true
tags:
  - Thrift
  - Java
---

愚蠢的人写的代码只有计算机能懂，优秀的程序员写的代码人人都懂。---- Martin Fowler, 1999

#### Thrift是什么

Thrift 是一个轻量级、语言无关、RPC调用框架的实现。它将调用抽象成三层：数据传输、数据序列化和应用层处理。并且提供了一个代码生成系统，只需要按照它提供的接口定义语言（IDL interface definition language）定义好接口，就可以用代码生成系统生成各种语言的代码。然后我们就可以使用生成的代码构建我们的客户端和服务端应用程序。

#### Thrift历史

Thrift 最初由Facebook开发，于2007年4月开源，2008年5月进入Apache 孵化器(Apache Incubator)，2010年10月成为Apache 顶级项目(Top-Level Project)。点击[Apache 孵化器指南][2]查看孵化器和顶级项目的区别

#### Thrift整体架构

![curl-img](../../img/2021-06-12-frame-thrift-one/thrift-layers.png)

#### Thrift 追求的目标

- **简单(Simplicity)** 代码简单易上手，没有不必要的依赖.

- **透明(Transparency)** 符合所有语言中最常见的习语.

- **一致(Consistency)** 特定于语言的功能属于扩展，而不是核心库.

- **性能(Performance)** 性能第一，优雅第二.

#### 实验环境搭建

系统信息:

```
~$ uname -a
Darwin 19.6.0 Darwin Kernel Version 19.6.0: Tue Jun 22 19:49:55 PDT 2021; root:xnu-6153.141.35~1/RELEASE_X86_64 x86_64
```

安装Thrift:

```
~$ brew install thrift
```

这里安装直接使用brew 进行，也可以自己[从源码编译][1]。
查看Thrift版本:

```
~$ thrift --version
Thrift version 0.15.0
```

如果使用老版本Thrift默认不会link到/usr/local/bin下。需要手动执行link。

```
~$ brew link thrift
```

克隆thrift仓库:

```
~$ git clone git@github.com:apache/thrift.git
~$ git checkout 0.15.0
cd tutorial
```

tutorial文件夹里是一个官方的例子，里面使用IDL定义了接口，并且有client和server的demo代码

使用thrift:

```
~$ thrift -r --gen java tutorial.thrift
```

执行完成，会自动创建一个gen-java的文件夹，里面就是生成的java代码。要生成其他语言的代码，只需要替换命令中的`java`。

查看其他语言支持情况:

```
~$ thrift --help
```

`Available generators (and options):` 段说明了其他语言的支持情况。由于现实跟理想的差距，各语言的支持情况会有不同，具体查看[语言和特性矩阵][3]

[1]: https://thrift.apache.org/docs/BuildingFromSource.html "Building from source"
[2]: https://alc-beijing.github.io/alc-site/post/apache-way/incubator-cook-book/ "Apache 孵化器指南"

[3]: https://thrift.apache.org/docs/Languages.html "语言和特性矩阵"
