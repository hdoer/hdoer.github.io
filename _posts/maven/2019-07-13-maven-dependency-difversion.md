---
title: "maven 测试和打包依赖不同版本"
layout: post
author: "Wentao Dong"
date: 2019-07-13 21:00:00
catalog: true
header-style: post
header-img: "img/city_night.png"
tags:
  - Java
  - Maven
  - Dependency
---

在排除依赖里我们讲到了几种排除依赖的方式，利用maven的路径最短原则，我们可以控制自己的服务依赖我们想要的版本。但还不够，有个特殊的需求：一般来说新加的函数都要写测试用例，但是有些测试用例由于要加载一个很大的模型，导致test很慢。为此我们需要让测试的时候使用测试的版本(小模型)，打包的时候使用另一个版本（大模型）

#### maven 版本

- 3.6.3

#### 依赖关系

A - B - C_larger，C是一个模型文件（二进制文件或文本文件）比如说：word2vec模型文件

C_larger是一个大模型文件

C_smaller是一个小模型文件

C_larger和C_smaller功能是一样的，只不过精度不同。仅依赖大模型是OK的，通常除了第一次运行test会比较慢，其他时候是没问题的。

#### 为什么要依赖不同版本

首先，虽然除了第一次运行test会比较慢外都没有问题，但是一般情况下运行test时我们不太关心模型的精度，加载过大的模型，显然会影响运行test的效率

其次，在一些场景下可能每次都会下载新的模型文件，大模型下载速度太慢，同样影响运行test的效率

#### 依赖不同版本解决方案

增加依赖 A - C_smaller，且scope改为test即可

```xml
<dependency>
    <groupId>com.company</groupId>
    <artifactId>C</artifactId>
    <version>${C_smaller.version}</version>
    <scope>test</scope>
</dependency>
```

如果还需要在指定 C_larger 的版本 也就是 A - C_larger. 可以这么写

```xml
<dependency>
    <groupId>com.company</groupId>
    <artifactId>C</artifactId>
    <version>${C_larger.version}</version>
</dependency>
<dependency>
    <groupId>com.company</groupId>
    <artifactId>C</artifactId>
    <version>${C_smaller.version}</version>
    <scope>test</scope>
</dependency>
```

`注意先写C_larger 再写C_smaller 不然测试的时候不会用C_smaller 原因是应用了同包声明覆盖原则`

如果你是要在顶级pom中声明dependencyManagement，下层pom中使用时不需要加version, 可以这么声明

```xml
<dependencyManagement>
<dependency>
    <groupId>com.company</groupId>
    <artifactId>C</artifactId>
    <version>${C_smaller.version}</version>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>com.company</groupId>
    <artifactId>C</artifactId>
    <version>${C_larger.version}</version>
</dependency>
</dependencyManagement>

下层要在pom中指定C_smaller版本，以便测试中使用C_smaller
<dependency>
    <groupId>com.company</groupId>
    <artifactId>C</artifactId>
</dependency>
<dependency>
    <groupId>com.company</groupId>
    <artifactId>C</artifactId>
    <version>${C_smaller.version}</version>
</dependency>
```
