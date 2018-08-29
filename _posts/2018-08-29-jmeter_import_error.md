---
layout: postlayout
title: jmeter导入jmx文件异常
categories: [jmeter]
tags: [jmeter]
---

不管是在学习框架还是其他东西时候最快的方式就是去查看别人的列子学习，然后不断的挖掘陌生点，根据陌生点去学习一个个新的知识点。

## 说明
之前有测试同事做过用jmeter来测试接口，然后问同事拿了个例子学习下，给我一个xxx.jmx 文件，然后进行本地导入jmeter，结果发现报错啦！

## 原因

报错信息如下：

```
2018-02-08 15:59:55,006 INFO o.a.j.g.a.Load: Loading file: C:\Users\Desktop\Testdemo\Testdemo.jmx
2018-02-08 15:59:55,006 INFO o.a.j.s.FileServer: Set new base='C:\Users\Desktop\Testdemo'
2018-02-08 15:59:55,006 INFO o.a.j.s.SaveService: Loading file: C:\Users\Desktop\Testdemo\Testdemo.jmx
2018-02-08 15:59:55,038 WARN o.a.j.g.a.Load: Unexpected error. java.lang.IllegalArgumentException: Problem loading XML from:'C:\Users\Desktop\Testdemo\Testdemo.jmx', missing class com.thoughtworks.xstream.converters.ConversionException: 
---- Debugging information ----
cause-exception     : com.thoughtworks.xstream.converters.ConversionException
cause-message       : 
first-jmeter-class  : org.apache.jmeter.save.converters.HashTreeConverter.unmarshal(HashTreeConverter.java:67)
class               : org.apache.jmeter.save.ScriptWrapper
required-type       : org.apache.jorphan.collections.ListedHashTree
converter-type      : org.apache.jmeter.save.ScriptWrapperConverter
path                : /jmeterTestPlan/hashTree/hashTree/com.atlantbh.jmeter.plugins.jsonutils.jsonpathextractor.JSONPathExtractor
line number         : 270
version             : 3.2 r17907481234567891011121314
```

主要原因：jmx文件中使用了JMeter的扩展jar包，但是JMeter中却没有相关jar包导致报错

## 解决

需要下载一个JAR包（jmeter-plugins-manager-1.3.jar，jmeter-plugins-manager.jar是jmeter的一个插件管理工具包）导进来就可以了。  插件官方下载地址：  https://jmeter-plugins.org/install/Install/，百度网盘下载（链接:https://pan.baidu.com/s/12Ylxr0PZTn7oe4lOHwEQLg  密码:5her）。下载完成后，将插件放到jmeter所在文件夹的lib/ext目录下，重启Jmeter即可。





参考：https://blog.csdn.net/yu1014745867/article/details/79291086