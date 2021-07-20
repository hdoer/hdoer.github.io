---
title: "command line - date"
# subtitle: "date 命令"
layout: post
author: "Wentao Dong"
date: 2021-07-14 21:00:00
catalog: false
header-style: post
header-img: "img/in-post/command/date-header.png"
tags:
  - 基础
  - command 
  - linux
  - mac
---

date 命令是工作中经常需要用到的命令，比如：

1. 自动化数据处理：在脚本中对日期进行格式化，然后用格式化后的字符串操作对应的文件
2. 日期与时间戳转换：为了阅读和日期比较我们经常需要在有格式的日期和时间戳之间进行转换

date 命令在linux 和 mac系统中都有，但是使用方式却不一样。一起来看看吧

#### Linux date

系统环境: 

```
ai_dev@iZ23a3yxjfdZ:~$ cat /etc/issue
Ubuntu 14.04.5 LTS \n \l

ai_dev@iZ23a3yxjfdZ:~$ uname -a
Linux iZ23a3yxjfdZ 3.13.0-150-generic #200-Ubuntu SMP Thu May 24 17:37:12 UTC 2018 x86_64 x86_64 x86_64 GNU/Linux
```

查看帮助:

```
ai_dev@iZ23a3yxjfdZ:~$ date --help
Usage: date [OPTION]... [+FORMAT]
   or: date [-u|--utc|--universal] [MMDDhhmm[[CC]YY][.ss]]
```

常用命令：

1. 无参数，打印当前日期时间

```
ai_dev@iZ23a3yxjfdZ:~$ date
Tue Jun 15 19:15:44 CST 2021
```

2. 格式化日期时间

```
ai_dev@iZ23a3yxjfdZ:~$ date '+%Y-%m-%d %H:%M:%S'
2021-06-15 19:21:34

全部参数请查看date命令帮助，常用参数如下:
%Y 年（如: 2021）
%m 月（两位数，如: 06）
%d 日（两位数，如: 15）
%H 时（两位数，如: 19）
%M 分（两位数，如: 21）
%S 秒（两位数，如: 34）
%D date; same as %m/%d/%y
%T time; same as %H:%M:%S
%n 换行
```

3. 日期加减

```
ai_dev@iZ23a3yxjfdZ:~$ date -d '-1 day'
Mon Jun 14 19:34:20 CST 2021

ai_dev@iZ23a3yxjfdZ:~$ date -d '+1 day'
Wed Jun 16 19:35:00 CST 2021

除了加减天数，还可以对其他日期时间字段加减，常用关键词如下:
year、month、day、hour、minute、second

还有不常用的，直接描述昨天、明天、下个星期二等，如:
ai_dev@iZ23a3yxjfdZ:~$ date -d 'tomorrow'
Wed Jun 16 19:39:06 CST 2021

tomorrow 可以换成如下关键词:
yesterday、next week、fortnight、next month
last monday、last tuesday、last wednesday、thursday、friday、saturday、sunday

指定起算时间，如6月1号后的第三周:
ai_dev@iZ23a3yxjfdZ:~$ date -d '6/1 3 week'
Tue Jun 22 00:00:00 CST 2021

将一种日期字符串转化为另一种日期字符串
ai_dev@iZ23a3yxjfdZ:~$ date -d '6/1/2021 21:00:00' '+%s'
1622552400

另外：
+ 等同于next, 所以date -d 'next day' 等于 date -d '+1 day'
- 等同于last, 所以date -d 'last day' 等于 date -d '-1 day'
```

4. 日期时间转换

```
ai_dev@iZ23a3yxjfdZ:~$ date +%s
1623760704

ai_dev@iZ23a3yxjfdZ:~$ date -d @1623760704
Tue Jun 15 20:38:24 CST 2021
```

5. 其他

```
显示文件的最后修改时间
ai_dev@iZ23a3yxjfdZ:~$ date -r test.txt
Tue Jun 15 20:18:48 CST 2021
```

#### Mac date

系统信息:

```
(base) [20:42:17] hdoer @ ~$ uname -a
Darwin bogon 19.6.0 Darwin Kernel Version 19.6.0: Tue Jan 12 22:13:05 PST 2021; root:xnu-6153.141.16~1/RELEASE_X86_64 x86_64
```

查看帮助:

```
~$ man date

SYNOPSIS
     date [-jRu] [-r seconds | filename] [-v [+|-]val[ymwdHMS]] ... [+output_fmt]
     date [-jnu] [[[mm]dd]HH]MM[[cc]yy][.ss]
     date [-jnRu] -f input_fmt new_date [+output_fmt]
     date [-d dst] [-t minutes_west]
```

常用命令：

1. 无参数，打印当前日期时间

```
~$ date
2021年 6月15日 星期二 20时50分54秒 CST
```

2. 格式化日期时间

```
~$ date '+%Y-%m-%d %H:%M:%S'
2021-06-15 20:54:27

全部参数请查看date命令帮助，常用参数如下:
%Y 年（如: 2021）
%m 月（两位数，如: 06）
%d 日（两位数，如: 15）
%H 时（两位数，如: 19）
%M 分（两位数，如: 21）
%S 秒（两位数，如: 34）
%D date; same as %m/%d/%y
%T time; same as %H:%M:%S
%n 换行
```

3. 日期加减

```
~$ date -v+1d
2021年 6月16日 星期三 21时06分15秒 CST

 ~$ date -v-1d
2021年 6月14日 星期一 21时06分28秒 CST

除了加减天数，还可以对其他日期时间字段加减，常用关键词如下:
y、m、w、d、H、M、S

指定具体时间，如6月1号的当前时间:
~$ date -v6m -v1d
2021年 6月 1日 星期二 21时10分20秒 CST

还有不常用的，直接描述下周一，如:
~$ date -v+monday
2021年 6月21日 星期一 21时12分56秒 CST

monday 可以换成如下关键词(前三个字母即可):
monday、tuesday、wednesday、thursday、friday、saturday、sunday

将一种日期字符串转化为另一种日期字符串
~$ date -j -f '%m %d %T' '06 1 21:00:00' '+%s'
1622552400
注意：-j 意思是不设置时间,原话： Do not try to set the date. 必须使用-j 才允许-f 后面加 '+' 格式化字符串
```

4. 日期时间转换

```
~$ date '+%s'
1623764139

~$ date -r 1623764139
2021年 6月15日 星期二 21时35分39秒 CST

将一种日期字符串转化为另一种日期字符串
~$ date -r 1623764139 '+%Y-%m-%d'
2021-06-15
```

5. 其他

```
显示文件的最后修改时间
~$ date -r test.txt
2021年 6月15日 星期二 21时37分17秒 CST
```

