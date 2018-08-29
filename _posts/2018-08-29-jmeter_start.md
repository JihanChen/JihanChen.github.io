---
layout: postlayout
title: jmeter安装
categories: [jmeter]
tags: [jmeter]
---

其实在很久之前就有想法去研究下如何做自动化测试，如何去提高自己的代码质量，或在某些场景下如何做接口测试，压力测试，找到系统瓶颈，但是每次都是止步不前，实在惭愧。最近突然沉下心来想好好学习接口测试这块。网上看到说JMeter具备了这些功能，就想学习下尝试下。

## 说明
Apache JMeter是Apache组织开发的基于Java的压力测试工具。用于对软件做压力测试，它最初被设计用于Web应用测试，但后来扩展到其他测试领域。 它可以用于测试静态和动态资源，例如静态文件、Java 小服务程序、CGI 脚本、Java 对象、数据库、FTP 服务器， 等等。JMeter 可以用于对服务器、网络或对象模拟巨大的负载，来自不同压力类别下测试它们的强度和分析整体性能。另外，JMeter能够对应用程序做功能回归测试，通过创建带有断言的脚本来验证你的程序返回了你期望的结果。为了最大限度的灵活性，JMeter允许使用正则表达式创建断言。

## 安装

### 第一步 下载
学习一个新的东东我们一般都是先下载工具然后安装，工欲善其事必先利其器。mac安装使用也比较简单，首先去JMeter的官网下载最新版本（官网地址：[http://jmeter.apache.org/download_jmeter.cgi](http://jmeter.apache.org/download_jmeter.cgi)）

下载最新的[apache-jmeter-4.0.tgz](http://mirrors.tuna.tsinghua.edu.cn/apache//jmeter/binaries/apache-jmeter-4.0.tgz)，打不开的可以网盘下载（链接:[https://pan.baidu.com/s/12Ylxr0PZTn7oe4lOHwEQLg](https://pan.baidu.com/s/12Ylxr0PZTn7oe4lOHwEQLg)  密码:5her）



### 第二步  解压
打开终端，命令cd到下载目录，对下载好的apache-jmeter-4.0.tgz进行解压, 命令也很简单

```
tar -zxvf apache-jmeter-4.0.tgz
```

### 第三步 启动

解压完之后，cd到解压目录下

```
 cd apache-jmeter-4.0
```

运行打开软件

```
./bin/jmeter
```

### 第四步 语言

打开的界面默认是英文的，所以想切换成中文，可以通过修改配置文件永久生效为中文。

1、在Jmeter的安装目录下的bin目录中找到 jmeter.properties这个文件，用文本编辑器打开。 

 2、大概在37行，找到：`#language=en`

\#Preferred GUI language. Comment out to use the JVM default locale's language.
\#language=en

3、将其修改为：`language=zh_CN`

\#Preferred GUI language. Comment out to use the JVM default locale's language.
language=zh_CN



参考：

          https://baike.baidu.com/item/Jmeter/3104456?fr=aladdin

          https://blog.csdn.net/GR9527/article/details/79876382

	  https://blog.csdn.net/him2014/article/details/79603887



## 结尾

jmeter的安装还是比较简单的，所以万事开头难也不是绝对的。


