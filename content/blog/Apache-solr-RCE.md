---
title: CVE-2019-0193 Apache Solr远程命令执行漏洞分析
date: 2019-08-07 10:38:17
tags: [vul,sec,Solr]
categories: Security
---

<script src="https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/pangu.js"></script>

### 0x01 概述

8月1日，Apache Solr官方发布了CVE-2019-0193漏洞预警，此漏洞存在于可选模块DataImportHandler中，DataImportHandler是用于从数据库或其他源提取数据的常用模块，该模块中所有DIH配置都可以通过外部请求的dataConfig参数来设置，由于DIH配置可以包含脚本，因此该参数存在安全隐患。

参考：<https://issues.apache.org/jira/browse/SOLR-13669>

### 0x02 环境搭建

选择Solr 8.1.1二进制版本进行分析和复现

下载地址：<https://archive.apache.org/dist/lucene/solr/8.1.1/>

调试命令

```shell
$ cd solr-8.1.1\server
$ java "-agentlib:jdwp=transport=dt_socket,server=y,suspend=n,address=9000" -Dsolr.solr.home="../example/example-DIH/solr/" -jar start.jar --module=http
```

IDEA新建远程调试即可

### 0x03 前置概念

solr支持从Dataimport导入自定义数据，dataconfig需要满足一定语法，参考

- <https://lucene.apache.org/solr/guide/6_6/uploading-structured-data-store-data-with-the-data-import-handler.html>

- <https://cwiki.apache.org/confluence/display/solr/DataImportHandler>

其中ScriptTransformer可以编写自定义脚本，支持常见的脚本语言如Javascript、JRuby、Jython、Groovy和BeanShell

> `ScriptTransformer`容许用脚本语言如Javascript、JRuby、Jython、Groovy和BeanShell转换，函数应当以行（类型为`Map<String,Object>`）为参数，可以修改字段。脚本应当写在数据仓库配置文件顶级的`script`元素内，而转换器属性值为`script:函数名`。

使用示例：

```xml
<dataconfig>
  <script><![CDATA[
    function f2c(row) {
      var tempf, tempc;
      tempf = row.get('temp_f');
      if (tempf != null) {
        tempc = (tempf - 32.0)*5.0/9.0;
        row.put('temp_c', temp_c);
      }
      return row;
    }
    ]]>
  </script>
  <document>

    <entity name="e1" pk="id" transformer="script:f2c" query="select * from X">
    </entity>
  </document>
</dataConfig>
```



Nashorn引擎

在Solr中解析js脚本使用的是Nashorn引擎，可以通过`Java.type`API在JavaScript中引用，就像Java的`import`一样，例如：

```java
var MyJavaClass = Java.type(`my.package.MyJavaClass`);

var result = MyJavaClass.sayHello('Nashorn');
print(result);
```

### 0x04 漏洞分析

Solr处理dataimport请求时，首先进入dataimport/DataImportHandler的handleRequestBody方法，当前请求的command为full-import，因此通过maybeReloadConfiguration重新加载配置

![1565157550593](https://blog-1252261399.cos.ap-beijing.myqcloud.com/images/1565157550593.png)

在maybeReloadConfiguration中通过params.getDataConfig()判断了post的数据(dataConfig)是否为空，如果不是则通过loadDataConfig来加载

![1565157730103](https://blog-1252261399.cos.ap-beijing.myqcloud.com/images/1565157730103.png)

随后在loadDataConfig中通过readFromXml方法解析提交的配置数据中的各个标签，比如`document`，`script`，`function`，`dataSource`等，传入的script自定义脚本即在此处被存入script变量，递归解析完所有标签构建出DIHConfiguration对象并返回。

获取到配置信息后通过this.importer.runCmd()方法处理导入过程

```java
this.importer.runCmd(requestParams, sw);
```

![1565158761569](https://blog-1252261399.cos.ap-beijing.myqcloud.com/images/1565158761569.png)

在doFullImport中，首先会创建一个DocBuilder对象，DocBuilder的主要功能是从给定配置中创建Solr文档，同时会记录一些状态信息。随后通过execute()方法会通过遍历Entity的所有元素来解析config结构，最终得到是一个EntityProcessorWrapper对象。EntityProcessorWrapper是一个比较关键的类，继承自EntityProcessor，在整个解析过程中起到重要的作用，可以参考

<https://lucene.apache.org/solr/8_1_1/solr-dataimporthandler/org/apache/solr/handler/dataimport/EntityProcessorWrapper.html>

在解析完config数据后solr会把最后更新时间记录到配置文件中，这个时间是为了下次进行增量更新的时候用的。接着通过this.dataImporter.getStatus()判断当前数据导入是“全部导入”还是“增量导入”，两个操作对应的方法分别为doDelta()和doFullDump()，此处的操作是full-import，因此调用doFullDump()

![1565165153995](https://blog-1252261399.cos.ap-beijing.myqcloud.com/images/1565165153995.png)

在doFullDump()中调用的是DocBuilder.buildDocument()方法，这个方法会为发送的配置数据的每一个processor做解析，当发送的entity中含有`Transformers`时，会进行相应的转换操作，例如转换成日期格式(DateFormatTransformer)、根据正则表达式转换(RegexTransformer)等，这次出现问题的是ScriptTransformer，可以根据用户自定义的脚本进行数据转换。由于脚本内容完全是用户控制的，当指定的script含有恶意代码时就会被执行，下面看一下Solr中如何执行javascript代码：

在读取EntityProcessorWrapper的每一个元素时，是通过epw.nextRow()调用的，它返回的是一个Map对象，进入EntityProcessorWrapper.nextRow方法

![1565166416846](https://blog-1252261399.cos.ap-beijing.myqcloud.com/images/1565166416846.png)

通过applyTransformer()执行转换，调用的是相应Transformer的transformRow方法

![1565166667450](https://blog-1252261399.cos.ap-beijing.myqcloud.com/images/1565166667450.png)

`ScriptTransformer`允许多种脚本语言调用，如Javascript、JRuby、Jython、Groovy和BeanShell等，transformRow()方法则会根据指定的语言来初始化对应的解析引擎，例如此处初始化的是scriptEngine，用来解析JavaScript脚本

![1565167039401](https://blog-1252261399.cos.ap-beijing.myqcloud.com/images/1565167039401.png)

Solr中默认的js引擎是Nashorn，Nashorn是于Java 8中用于取代Rhino（Java 6，Java 7）的JavaScript引擎，在js中可以通过`Java.type`引用Java类，就像Java的`import`一样，此处就可以通过这个语法导入任意Java类。

随后通过反射调用自定义的函数并执行，例如通过java.lang.Runtime执行系统命令

![1565166706725](https://blog-1252261399.cos.ap-beijing.myqcloud.com/images/1565166706725.png)

整个漏洞就是因为可以通过`<script>`标签指定ScriptTransformer，而在这个标签内可以导入任意的java类，Solr也并没有对标签内容做限制，导致可以执行任意代码。

![1565167939907](https://blog-1252261399.cos.ap-beijing.myqcloud.com/images/1565167939907.png)

调用栈情况

![1565161745688](https://blog-1252261399.cos.ap-beijing.myqcloud.com/images/1565161745688.png)



### 0x05 补充

值得注意的是，官方给出的临时修复方案并不能缓解漏洞，当把相应的index core的配置文件置为空时，dataimport的时候只是获取不到默认的配置，但是依然能够通过这个接口发送PoC，漏洞也依然能够触发，解决办法是把相应配置文件中的dataimport requestHandler全部注释并重启Solr服务器，才能彻底关闭这个接口缓解漏洞。



<script>pangu.spacingPage();</script>





