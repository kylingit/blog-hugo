---
title: Apache Solr Velocity模板注入漏洞分析
date: 2019-11-05 09:40:22
tags: [vul,sec,Solr]
categories: Security
---

<script src="https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/pangu.js"></script>

### 0x01 概述

10月30日，研究员S00pY在GitHub发布了Apache Solr Velocity模版注入远程命令执行的poc，该漏洞通过设置资源加载属性，利用`VelocityResponseWriter`插件执行自定义模板，进而进行远程代码执行，危害较大，下面是分析过程。

### 0x02 环境搭建

选择Solr 8.2.0二进制版本进行分析和复现

下载地址：<https://archive.apache.org/dist/lucene/solr/8.2.0/>

调试命令

```shell
$ cd solr-8.2.0\server
$ java "-agentlib:jdwp=transport=dt_socket,server=y,suspend=n,address=9000" -Dsolr.solr.home="../example/example-DIH/solr/" -jar start.jar --module=http
```

IDEA新建远程调试即可

### 0x03 前置概念

`VelocityResponseWriter`(Velocity响应编写器)是 contrib/velocity 目录中可用的可选插件。当使用诸如 “_default”、“techproducts” 和 “example / files” 等配置时，它为浏览用户界面提供动力。

必须添加它的 JAR 和依赖项（通过<lib>或 solr/home lib 包含），并且必须在 solrconfig.xml 注册，默认已经注册

![1572507926900](https://blog-1252261399.cos.ap-beijing.myqcloud.com/images/1572507926900.png)

其中有一个属性`params.resource.loader.enabled`，默认是`false`，需要手动开启

该参数表示允许加载程序在 Solr 请求参数中指定模板，例如：

```
http://localhost:8983/solr/gettingstarted/select?q=\*:*&wt=velocity&v.template=xxx&v.template.xxx=CUSTOM%3A%20%23core_name
```

`v.template=xxx`表示创建一个名为“xxx”的模板，`v.template.xxx`则是模板内容

当这个属性设置为`true`时用户就可以传入任意模板内容进行模板注入，从而执行任意命令

### 0x04 漏洞分析

#### 设置`params.resource.loader.enabled`属性

在Solr的Web.xml文件中能看到所有的请求都交给`org.apache.solr.servlet.SolrDispatchFilter`来处理

![1572508606062](https://blog-1252261399.cos.ap-beijing.myqcloud.com/images/1572508606062.png)

具体的则是其中的`doFilter()`方法。在对路由经过初步处理后，进行两个关键操作：

```java
HttpSolrCall call = this.getHttpSolrCall(request, response, retry);
//...
SolrDispatchFilter.Action result = call.call();
```

初始化一个`HttpSolrCall`对象后调用它的`call()`方法，在`call()`方法中会对路由中具体的组件初始化出对应的handler，再由这个handler去调用这个组件的各个方法

在Solr 8.2.0中具体的路由有37个，每一类都有对应的handler，都在`org.apache.solr.handler`中定义，例如`solr/solr/get`对应的hendler为`RealTimeGetHandler`

![1572515458181](https://blog-1252261399.cos.ap-beijing.myqcloud.com/images/1572515458181.png)

![1572516210537](https://blog-1252261399.cos.ap-beijing.myqcloud.com/images/1572516210537.png)

而`/solr/solr/config` 由`SolrConfigHandler`来分别处理GET和POST请求

```java
SolrConfigHandler.Command command = new SolrConfigHandler.Command(req, rsp, httpMethod);
if ("POST".equals(httpMethod)) {
    if (configEditing_disabled || this.isImmutableConfigSet) {
        String reason = configEditing_disabled ? "due to disable.configEdit" : "because ConfigSet is immutable";
        throw new SolrException(ErrorCode.FORBIDDEN, " solrconfig editing is not enabled " + reason);
    }

    try {
        command.handlePOST();
    } finally {
        RequestHandlerUtils.addExperimentalFormatWarning(rsp);
    }
} else {
    command.handleGET();
}
```

私有类`Command`会对当前路由的webapp和path做一个切分，对于POST请求，分别会通过`SolrConfigHandler.Command#handlePOST()`方法来处理

![1572509660264](https://blog-1252261399.cos.ap-beijing.myqcloud.com/images/1572509660264.png)

接着调用`SolrConfigHandler.Command#handleCommands()`，Solr中`Config API`对应的实现都是由这个方法来完成的，如`set-property`、`unset-property`等

![1572516461804](https://blog-1252261399.cos.ap-beijing.myqcloud.com/images/1572516461804.png)

此处主要关注更新配置的参数

从[文档](https://lucene.apache.org/solr/guide/8_2/config-api.html#basic-commands-for-components)可以了解对于`responsewriter`的操作有下面三个

- `add-queryresponsewriter`
- `update-queryresponsewriter`
- `delete-queryresponsewriter`

![1572516621585](https://blog-1252261399.cos.ap-beijing.myqcloud.com/images/1572516621585.png)

代码中也能看到对操作名称按`-`进行分割提取出对应操作，然后由`updateNamedPlugin()`方法来完成配置文件的创建/覆盖操作，具体跟入看一下

在`updateNamedPlugin()`中有个`verifyClass`的调用，当传入参数没有设置`runtimeLib`时会去创建class字段指定的类，所以当我们传入`VelocityResponseWriter`时，会在其初始化的时候写入对应的参数

![1572518046827](https://blog-1252261399.cos.ap-beijing.myqcloud.com/images/1572518046827.png)

然后返回到`handleCommands()`中把配置写入到`configoverlay.json`文件

![1572518122216](https://blog-1252261399.cos.ap-beijing.myqcloud.com/images/1572518122216.png)

因此，通过`config api`可以重新设置`VelocityResponseWriter`的属性，为下一步加载模板提供入口

三种命令的区别如下：

> add- 命令都会将新配置添加到configoverlay.json，这将覆盖solrconfig.xml组件中的任何其他设置；
> update- 命令覆盖configoverlay.json中的现有设置；
> delete-命令从configoverlay.json中删除设置



#### 注入自定义模板

在`SolrDispatchFilter`中有有一个枚举类`Action`，定义了每个handler的所属的操作，通过ConfigAPI更新配置时，当前的action是`PROCESS`，因此会进入`HttpSolrCall.call()`的`PROCESS`分支

![1572576900618](https://blog-1252261399.cos.ap-beijing.myqcloud.com/images/1572576900618.png)

之后通过`QueryResponseWriterUtil.writeQueryResponse()`进入`VelocityResponseWriter.write`，在这个方法中完成`Velocity`的解析

![1572577848829](https://blog-1252261399.cos.ap-beijing.myqcloud.com/images/1572577848829.png)

首先会初始化一个解析模板的引擎`VelocityEngine`，在创建引擎的过程中会检查是否允许参数资源加载，这也就是第一个请求设置的`params.resource.loader.enabled`属性值。由于`solr.resource.loader.enabled`默认是开启的，所以此处只需要设置params的值

![1572577993537](https://blog-1252261399.cos.ap-beijing.myqcloud.com/images/1572577993537.png)

之后通过`Template.getTemplate()`设置自定义模板，然后进入`Template.merge()`进入AST解析，在解析过程中会调用到`ASTMethod.execute()`方法，这个流程与之前披露的CVE-2019-11581 JIRA模板注入漏洞是一样的，不再赘述，详细可以参考[CVE-2019-11581 ATLASSIAN JIRA 未授权模板注入漏洞分析](https://kylingit.com/blog/cve-2019-11581-atlassian-jira%E6%9C%AA%E6%8E%88%E6%9D%83%E6%A8%A1%E6%9D%BF%E6%B3%A8%E5%85%A5%E6%BC%8F%E6%B4%9E%E5%88%86%E6%9E%90/)

>回过头看一下`Velocity`渲染的大致流程：
>
>> Velocity 渲染引擎首先磁盘加载模板文件到内存，然后解析模板模板文件为 AST 结构，并对 AST 中每个节点进行初始化，第二次加载同一个模板文件时候如果开启了缓存则直接返回模板资源，通过使用资源缓存节省了从磁盘加载并重新解析为 AST 的开销。
>
>而`ASTMethod.execute()`方法设计之初是在`Velocity parse`解析模板的过程中，通过反射调用相关方法完成正常模板渲染动作，例如获取背景颜色、获取 text 内容、获取页面编码等，但当此处攻击者传入精心构造的数据后，利用反射执行了`java.lang.Runtime.getRuntime`，成功达到命令执行的目的

调用栈

![1572575433979](https://blog-1252261399.cos.ap-beijing.myqcloud.com/images/1572575433979.png)

PoC

![1572578779708](https://blog-1252261399.cos.ap-beijing.myqcloud.com/images/1572578779708.png)



参考

- https://lucene.apache.org/solr/guide/8_2/config-api.html#basic-commands-for-components
- https://github.com/veracode-research/solr-injection#7-cve-2019-xxxx-rce-via-velocity-template-by-_s00py



<script>pangu.spacingPage();</script>




