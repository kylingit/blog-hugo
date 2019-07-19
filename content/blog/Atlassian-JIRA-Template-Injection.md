---
title: CVE-2019-11581 Atlassian Jira未授权模板注入漏洞分析
date: 2019-07-17 10:52:08
tags: [vul,sec,Drupal]
categories: Security
---

<script src="https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/pangu.js"></script>

### 0x01 概述

7月10日，`Atlassian`官方发布安全公告，修复了`Jira Server`和`Jira Data Center`的一个模板注入漏洞，CVE编号`CVE-2019-11581`

公告地址：<https://confluence.atlassian.com/jira/jira-security-advisory-2019-07-10-973486595.html>



### 0x02 环境搭建

使用`atlas-debug`调试

1. 下载安装Atlassian SDK，[地址](https://developer.atlassian.com/server/framework/atlassian-sdk/install-the-atlassian-sdk-on-a-windows-system/)；
2. `atlas-create-jira-plugin`创建一个插件，[参考](https://developer.atlassian.com/server/framework/atlassian-sdk/create-a-helloworld-plugin-project/)；
3. `atlas-debug`开启调试，http端口2990，调试端口5005；
4. IDEA打开MyPlugin，把`WEB-INF/classes`和`WEB-INF/lib`加入library；
5. 新建Remote调试；

其他：

1. 如果没有设置%JAVA_HOME%可以通过`SET JAVA_HOME=d:\jdk1.8`设置；
2. 默认不开启电子邮件发送，通过`atlas-debug --jvmargs -Datlassian.mail.senddisabled=false`开启；



### 0x03 漏洞分析

#### 第一部分：注入代码并生成邮件

![1563246860552](https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/images/1563246860552.png)

`post`的数据通过`JiraSafeActionParameterSetter->setActionProperty()`方法

![1563271320369](https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/images/1563271320369.png)

通过反射调用到`ContactAdministrators.setSubject()`方法，把`ContactAdministrators`对象的`subject`属性设置为传入的`subject`

随后通过`ContactAdministrators.doExecute()`调用`send()`方法，在这个方法中会查找系统中已激活的管理员，通过`this.sendTo(administrator)`将邮件发送给该管理员

![1563247236827](https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/images/1563247236827.png)

在`sendTo()`流程中，`Jira`需要通过`EmailBuilder()`方法创建一个邮件队列对象，随后将该对象放入邮件发送队列中。由于队列等待原因，所以触发`payload`可能需要等待一段时间，并且当邮件发送失败时系统会继续尝试发送邮件，所以`payload`可能会触发多次。

![1563247471810](https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/images/1563247471810.png)

创建队列的方法有点长，精简一下就是这个样子

```java
MailQueueItem item = (new EmailBuilder()).withSubject(this.subject).withBodyFromFile().addParameters().renderLater();
```

通过`EmailBuilder`的`withSubject()`方法，创建一个`TemplateSources$fragment`对象，参数即是我们传入的`payload`，随后调用`renderLater()`方法创建出`EmailBuilder`对象，再将该对象作为参数传递给`RenderingMailQueueItem`类，`RenderingMailQueueItem`的继承关系是如下图，于是最终创建出一个`MailQueueItem`对象，并将该对象放入邮件发送队列。

![1563248388715](https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/images/1563248388715.png)



#### 第二部分：发送邮件

当我们把`payload`注入到模板中之后，邮件进入待发送队列，Jira中处理邮件队列的具体流程如下：

通过模板引擎`(getTemplatingEngine)`生成一个`Velocity`模板，通过`applying()`方法生成`RenderRequest`对象，之后根据该对象成员变量`source`的类型，调用不同的方法解析模板，漏洞的产生正是由于这个差异造成的，下面详细分析一下。

首先进入`RenderingMailQueueItem().send()`方法，调用`this.emailRenderer.render()`，随后调用

```java
this.getTemplatingEngine().render(this.subjectTemplate).applying(contextParams).asPlainText();
```

这个过程中前面是为了获取模板解析引擎`（VelocityTemplatingEngine）`并传入主题模板（此处为payload数据），通过`applying()`方法创建`VelocityContext`对象并把`payload`赋值给成员变量`source`

![1563267497538](https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/images/1563267497538.png)

随后重写了抽象类`StringRepresentation`的`with()`方法，在`with()`方法中调用了`asPlainText()`方法

```java
DefaultRenderRequest.this.asPlainText(sw)
```

`asPlainText()`的作用是通过`Velocity`模板引擎解析模板，其中的调用链是

```
toWriterImpl()->writeEncodedBodyForContent()->evaluate()
```

而在`evaluate()`方法中生成了`AST`结构，随后通过反射调用传入的`payload`，完成代码执行。

![1563245970455](https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/images/1563245970455.png)

`asPlainText()`之后的调用栈如下

![1563246607016](https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/images/1563246607016.png)

在处理完Object模板后会调用父类`SingleMailQueueItem`的send()方法，通过`smtpMailServer.sendWithMessageId()`发送邮件，由于没有正确配置`SMTP`服务会抛出异常，但在连接`SMTP`服务之前漏洞已经触发了，控制台也能看到`MailQueue`执行的过程。

![1563257377543](https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/images/1563257377543.png)

### 思考

上述漏洞流程走完了，但还有一个关键问题没有解决：为什么邮件主题`Subject`会被解析成`AST`结构并被执行呢？按照正常发送反馈的逻辑，一封邮件的主题（字符串）似乎没有必要解析成`AST`，导致差异的原因是什么？

发送一封正常的“联系管理员”邮件，走一遍流程

![1563268202907](https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/images/1563268202907.png)

![1563268493868](https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/images/1563268493868.png)

对比一下两个处理流程，当发送正常反馈时，`writeEncodedBody()`中调用的是`this.getVe().mergeTemplate`，通过`Velocity`引擎的`ClasspathResourceLoader()`类的`getResourceStream()`方法加载模板文件，此处的模板是`templates/email/html/contactadministrator.vm`，随后还会进行`header`、`footer`等正常加载流程，最终渲染出整个页面。而发送`payload`时，通过asPlainText()创建出TemplateSource$Fragment对象，再通过DefaultRenderRequest构造方法把`source`成员变量赋值为这个`Fragment`对象，于是进入第一个分支，调用的是`this.getVe().evaluate()`，最终调用`ASTMethod.execute()`，这正是前面说的差异性导致的两个不同处理逻辑。

回过头看一下`Velocity`渲染的大致流程：

>Velocity渲染引擎首先磁盘加载模板文件到内存，然后解析模板模板文件为AST结构，并对AST中每个节点进行初始化，第二次加载同一个模板文件时候如果开启了缓存则直接返回模板资源，通过使用资源缓存节省了从磁盘加载并重新解析为AST的开销。

而`ASTMethod.execute()`方法设计之初是在`Velocity parse`解析模板的过程中，通过反射调用相关方法完成正常模板渲染动作，例如获取背景颜色、获取text内容、获取页面编码等，但当此处攻击者传入精心构造的数据后，利用反射执行了`java.lang.Runtime.getRuntime`，成功达到命令执行的目的，漏洞利用十分精巧。

补充：严格意义上讲这不属于一个模板注入漏洞，因为代码并没有“注入”到某一个模板中，漏洞触发过程是在解析模板文件之前，利用的是解析模板的过程，恶意代码并没有真正落地也没有插入到某个模板里面，所以称之为“代码注入”也许更合适。

### 参考

- <https://confluence.atlassian.com/jira/jira-security-advisory-2019-07-10-973486595.html>
- <https://developer.atlassian.com/server/framework/atlassian-sdk/atlas-debug/>
- <https://developer.atlassian.com/server/framework/atlassian-sdk/create-a-helloworld-plugin-project/>
- <http://ifeve.com/velocity%E5%8E%9F%E7%90%86%E6%8E%A2%E7%A9%B6/>



<script>pangu.spacingPage();</script>



