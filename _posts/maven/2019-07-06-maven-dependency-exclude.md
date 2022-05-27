---
title: "maven 依赖排除"
layout: post
author: "Wentao Dong"
date: 2019-07-06 21:00:00
catalog: true
header-style: post
header-img: "img/city_night.png"
tags:
  - Java
  - Maven
  - Dependency
---

Java 开发最方便的是想要啥功能基本都有包。但是引入的包太多，怎么管理起来成了不得不解决的问题，然后大舅星 [maven](https://maven.apache.org/) 出现了。使用maven的过程中我们会遇到很多问题，比如今天要讲的依赖排除

#### maven 版本

- 3.6.3

#### 为什么要排除某些依赖

java 开发中需要依赖各种各样的包，有些包我们不需要，有些包我们不能要。

* 不需要:  比如我们需要使用包A中的某个类，所以把包A依赖了进来, 可是包A 依赖了一个模型文件，这个文件还特别大，但是我们并不需要

* 不能要：我们最头疼的是log了，五花八门的log 依赖，有时候会造成循环。如我们用的是log4j来管理日志，通常为了实现统一日志输出，我们会使用slf4j + slf4j-log4j + log4j 这种方式，打log仅调用slf4j的api 就可以了，具体使用什么东西管理日志，就看使用的绑定器了。当我们依赖的其他包使用的不是slf4j时，我们使用 ***桥接器（如： jcl-over-slf4j.jar）*** 将 日志流导向到slf4j。但是假如我们依赖的包依赖了 log4j-over-slf4j 会出现什么情况呢？会形成死循环：日志-> slf4j -> log4j -> slf4j.

#### 如何排除这些依赖

1. 使用 maven 的 exclusions
   
   ```xml
   <dependency>
       <groupId>com.company</groupId>
       <artifactId>a</artifactId>
       <version>${a.version}</version>
       <exclusions>
           <exclusion>
               <groupId>com.company</groupId>
               <artifactId>b</artifactId>
           </exclusion>
       </exclusions>
   </dependency>
   ```

2. 使用插件：[maven-enforcer-plugin](http://maven.apache.org/plugins/maven-enforcer-plugin/)
   
   这个其实是预防功能，防止使用了方法1排除了依赖，后来引入的其他依赖再次把已经排除的包依赖进来。该插件可以控制使用的maven版本、java版本甚至使用的操作系统，当然也能控制依赖的包。效果就是当不满足我们的配置，比如不让依赖中出现commons-lang 的包却出现了，可以直接让其编译失败。提醒开发者修改依赖
   
   ```xml
   <plugin>  
       <groupId>org.apache.maven.plugins</groupId>  
       <artifactId>maven-enforcer-plugin</artifactId>
       <version>3.0.0-M2</version>
       <executions>  
         <execution>  
           <id>enforce-versions</id>  
           <goals>  
             <goal>enforce</goal>  
           </goals>  
           <configuration>  
             <rules>
               <bannedPlugins>
                 <!-- will only display a warning but does not fail the build. -->
                 <level>WARN</level>
                 <excludes>
                   <exclude>org.apache.maven.plugins:maven-verifier-plugin</exclude>
                 </excludes>
                 <message>Please consider using the maven-invoker-plugin (http://maven.apache.org/plugins/maven-invoker-plugin/)!</message>
               </bannedPlugins>
               <requireMavenVersion>  
                 <version>3.1.0</version>  
               </requireMavenVersion>  
               <requireJavaVersion>  
                 <version>1.8</version>  
               </requireJavaVersion>
               <requireOS>
                 <family>unix</family>
               </requireOS>
             </rules>  
           </configuration>  
         </execution>  
         <execution>  
           <id>enforce-banned-dependencies</id>  
           <goals>  
             <goal>enforce</goal>  
           </goals>  
           <configuration>  
             <rules>  
               <bannedDependencies>  
                 <excludes>  
                   <exclude>junit:junit</exclude>
                   <exclude>commons-lang:commons-lang</exclude>
                 </excludes>  
                 <includes>  
                   <include>junit:junit:4.8.2:jar:test</include> 
                 </includes>  
               </bannedDependencies>  
             </rules>  
             <fail>true</fail>  
           </configuration>  
         </execution>  
       </executions>  
   </plugin>
   ```

3. 利用maven依赖管理原则排除依赖
   
   maven 依赖管理原则：
   
   1. 路径最短原则: 如果依赖链为 A - B - C1 和 A - B - D - C2；则A 最终依赖的C的版本为 C1
   
   2. 不同包声明优先原则: 如果依赖链为 A - B - C1 和 A - D - C2 但是在A中先声明了B; 则A最终依赖的C的版本为C1
   
   3. 同包声明覆盖原则：如果依赖链为 A - C1 和 A - C2 就是A 依赖了C 的两个版本，但是C1 声明在前，则A 最终依赖的C的版本为C2
   
   所以根据这个特性，如果我们要排除C1 可以在A中直接声明依赖C1，并设置scope为 provided. 注意在最终包里使用，spring-boot项目的话，直接在server module里排除
   
   ```xml
   <dependency>  
       <groupId>com.company</groupId>  
       <artifactId>C</artifactId>  
       <version>1.0</version>  
       <scope>provided</scope>  
   </dependency>
   ```

4. 发布空包
   
   如果有自己的maven仓库，可以打包一个空的C1 推到仓库中，这样需要依赖这个包的时候，从自己的仓库中拉取，就不会有依赖问题了
