---
layout: post
title: MySQL 历史SQL语句，性能问题查找
categories: MySQL
description: MySQL 历史SQL语句，性能问题查找
keywords: MySQL history SQL
---

经常会碰到应用反馈上周出现过SQL性能问题，离上周已经过了好几天，怎么查，太难了。。。
对于MySQL了解来说排查这种问题，基本3中思路：

应用问题日志信息；
MySQL的慢日志可以，慢日志开了吗，多少秒记录指标；
MySQL的binlog应该会有记录，binlog保留多少天；
MySQL的监控指标；
以上4种都是常用的手段，除了这些方式，还能不能抓到特定的SQL语句，是否存在性能问题。 如：disk,lock,wait,join,scan,index,rows 等。

得到答案之前，先了解一下MySQL企业版怎么去解决这样的问题。


## 企业版SQL语句性能分析方式：
MySQL Enterprise Monitor(简称EM）是怎么做到sql语句的性能监控。官网提供一个月使用企业版功能，可自行下载研究。
EM对于数据库性能指标（QRTi表示查询响应时间）:
![](/images/posts/java/idea-unsupported-java-version.png)
有不少网友提到的一个措施是修改 IDEA 自身运行的 Runtime，即 JDK 版本。也决定试一下看看效果，于是安装了 `Choose Runtime` 插件，然后将默认的 JetBrains Runtime 由 IDEA 自带 JDK 11 换成了我自己安装的 JDK1.8.0_271，然后……IDEA 就再也起不来了，启动就报如下这个错误：

![](https://kevin-cui.github.io/mysqlstone/images/posts/mysql/20210604-01.png)

```
Unsupported Java Version

Java 11 or newer is required to run the IDE.
Please contact support at https://jb.gg/ide/critical-startup-errors

Your JRE: 1.8.0_271-b09 x86_64 (Oracle Corporation)
/Library/Java/JavaVirtualMachines/jdk1.8.0_271.jdk/Contents/Home/jre
```

## 解决方法

1. 想办法进设置 Runtime 的地方，将配置再改回来。

    但并没有找到办法进设置。失败

2. 想办法找到存储这个配置项的配置文件，手动修改回来。

    在网上搜、按经验找了 `~`、`~/Library/Preferences` 等文件夹，均未找到正在使用的 2020.3 版本的配置文件。失败

3. 此时留意到以上错误提醒里有个链接，打开链接寻找线索。

    - 打开 <https://jb.gg/ide/critical-startup-errors>，在该页面并未直接找到答案，但在侧边栏发现了线索链接。
    - 跳转到 [Selecting the JDK version the IDE will run under](https://intellij-support.jetbrains.com/hc/en-us/articles/206544879-Selecting-the-JDK-version-the-IDE-will-run-under)，在正文的 macOS 部分，提到了如果配置过 IDE JDK Version，会保存在配置文件目录下的 `<product>.jdk` 文件里，并提供了配置文件目录相关的链接。
    - 跳转到 [Directories used by the IDE to store settings, caches, plugins and logs](https://intellij-support.jetbrains.com/hc/en-us/articles/206544519)，可以找到 macOS 下 IDEA 2020.3 的配置文件路径 idea.config.path 为 `~/Library/Application Support/JetBrains/IntelliJIdea2020.3`，打开该目录，果然发现了 idea.jdk 文件。
    - 将 idea.jdk 文件删除，重新打开 IDEA，问题解决。

## 小结

当遇到问题时，最应该关注的是错误提示里的信息，很可能解决方案或线索就在里面。

如果以上解决不了问题，在官方文档/网站等渠道寻找解决方案会比盲目全网搜索更精准。如 [Configuration directory](https://www.jetbrains.com/help/idea/tuning-the-ide.html#config-directory) 这个链接里就清楚地描述了 IntelliJ IDEA 配置文件的存放位置。
