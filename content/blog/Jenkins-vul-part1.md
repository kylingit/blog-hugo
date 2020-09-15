---
title: Jenkins 路由解析及沙箱绕过漏洞分析报告(上)
date: 2020-09-15 16:42:32
tags: [sec,Java,Jenkins,Groovy]
categories: Security
---

<script src="https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/pangu.js"></script>



# Jenkins 路由解析及沙箱绕过漏洞分析报告(上)

![](https://jenkins.io/images/logo-title-opengraph.png)



## 简介

本报告主要研究Jenkins的路由解析机制和Groovy沙箱绕过带来的安全问题，梳理Jenkins官方2018-2019年以来涉及沙箱绕过的安全更新，探讨Java沙箱在Java应用中的安全性。由于篇幅较长，分为上下两篇发表，文中疏漏之处还请批评指正。

Jenkins是一个开源软件项目，是基于Java开发的一种持续集成工具，用于监控持续重复的工作，旨在提供一个开放易用的软件平台，使软件的持续集成变成可能。Jenkins的目的是持续、自动化地构建/测试软件项目以及监控软件开发流程，快速问题定位及处理，提升开发效率。

Script Security插件是Jenkins的一个安全插件，可以集成到Jenkins各种功能插件中。它主要支持两个相关系统：脚本批准和Groovy沙箱，分别用来管控脚本是否允许执行以及将脚本限制在安全环境下执行，避免带来不可控风险。

### 环境搭建

1. 下载相应版本的war包

    地址：https://updates.jenkins-ci.org/download/war/

2. 设置环境变量JENKINS_HOME

    `set JENKINS_HOME=D:\Jenkins\jenkins_2.137`

3. 加上调试选项并运行

    `java -agentlib:jdwp=transport=dt_socket,server=y,suspend=n,address=5005 -jar jenkins_2.137.war --httpPort=8082`

4. 安装插件

    国内镜像地址：https://mirrors.tuna.tsinghua.edu.cn/jenkins/updates/update-center.json

    Jenkins在安装过程中会自动下载部分插件的最新版，这部分可以先跳过，再在后台上传特定版本的插件（`.hpi`文件）进行安装，然后重启Jenkins完成安装




## 动态路由机制

首先从WEB-INF/web.xml入手看看Jenkins如何处理路由，可以看到所有请求都交给org.kohsuke.stapler.Stapler，具体是由Stapler:service()方法来处理

![1568171738754](https://blog-1252261399.cos.ap-beijing.myqcloud.com/images/1568171738754.png)

![1568171941279](https://blog-1252261399.cos.ap-beijing.myqcloud.com/images/1568171941279.png)

在service方法中主要调用的是invoke方法，两处调用的区别是invoke的第3和第4个参数不同，分别是根节点root和url路径，在调用之前判断了url路径，如果是`/$stapler/bound/`开头，则把根节点设置为`boundObjectTable`，否则通过`this.webApp.getApp()`把根节点设置为`hudson.model.Hudson`

跟进invoke方法

![1568172501683](https://blog-1252261399.cos.ap-beijing.myqcloud.com/images/1568172501683.png)

![1568172552611](https://blog-1252261399.cos.ap-beijing.myqcloud.com/images/1568172552611.png)

调用的是`Stapler#tryInvoke()`方法，tryInvoke()方法中对node类型（也就是一开始的root）进行了判断，按先后顺序分别处理三种情况

- StaplerProxy
- StaplerOverridable
- StaplerFallback

这三种情况的具体区别可以参考Jenkins关于[路由请求](https://jenkins.io/zh/doc/developer/handling-requests/routing/)的文档

这里我们关注中间获取metaClass和调用dispatch的过程

![1568172929817](https://blog-1252261399.cos.ap-beijing.myqcloud.com/images/1568172929817.png)

通过传入`/securityRealm/user/admin/`动态调试来跟踪理解

### 初始化metaClass

WebApp中会缓存一个classMap存放MetaClass，无对应缓存则通过 

```java
mc = new MetaClass(this, c); 
```

进行初始化，这个过程发生在Jenkins刚启动没有缓存时，当建立缓存后则直接从classMap获取相应的MetaClass

```java
MetaClass mc = (MetaClass)this.classMap.get(c);
```

![1568188301839](https://blog-1252261399.cos.ap-beijing.myqcloud.com/images/1568188301839.png)

 在MetaClass的构造方法中，会再次调用其父类的`getMetaClass()`方法，直到父类为空为止

![1568188747361](https://blog-1252261399.cos.ap-beijing.myqcloud.com/images/1568188747361.png)

而此时的kclass为一开始传入的`hudson.model.Hudson`，Hudson继承关系如下图

![1568193610000](https://blog-1252261399.cos.ap-beijing.myqcloud.com/images/1568193610000.png)

因此getMetaClass一直调用到class java.lang.Object，然后进行`buildDispatchers()`方法并层层返回，因此整个初始化metaClass的过程是一个不断寻找继承树并递归调用buildDispatchers的过程，而buildDispatchers的功能就是提取该类中的所有函数信息存储在`MetaClass.dispatchers`中，作为后续与url的映射关系

buildDispatchers()方法按顺序调用如下：

- this.registerDoToken(node)
    - do<token>(…)
- node.methods.prefix("js").iterator()
    - js<token>(…)
- node.methods.annotated(JavaScriptMethod.class).iterator()
    - @JavaScriptMethod annotation
- node.methods.prefix("get")
    - get<token>()
    - get<token>(String)
    - get<token>(Int)
    - get<token>(Long)
- getMethods.signature(new Class[]{StaplerRequest.class}).iterator()
    - get<token>(StaplerRequest)
- getMethods.signatureStartsWith(new Class[]{String.class}).name("getDynamic").iterator()
    - getDynamic(...)
- node.methods.name("doDynamic").iterator()
    - doDynamic(…)

这相当于规定了一个函数命名规则，只要符合这个规则的方法都能被访问到。

注意此过程中，大部分dispatchers添加的都是`NameBasedDispatcher`对象，除了如下几类：

- DirectoryishDispatcher (url路径相关，如`/`、`?`、`../`等)
- HttpDeletableDispatcher (`DELETE`方法)
- IndexDispatcher (`doIndex(...)`)
- Dispatcher (`getDynamic(…)` `doDynamic(…)`)

其中`js<token>(…)`对应的`JavaScriptProxyMethodDispatcher`继承自`NameBasedDispatcher`

![1568196821681](https://blog-1252261399.cos.ap-beijing.myqcloud.com/images/1568196821681.png)

hudson.model.Hudson经过递归buildDispatchers，缓存下的dispatchers有220个，根据上面的注意点，其中大部分方法会调用到`NameBasedDispatcher#dispatch()`

![1568196931745](https://blog-1252261399.cos.ap-beijing.myqcloud.com/images/1568196931745.png)

![1568195521387](https://blog-1252261399.cos.ap-beijing.myqcloud.com/images/1568195521387.png)



### 递归解析路由

回到org.kohsuke.stapler.Stapler#tryInvoke()，路径`/securityRealm/`对应的是`hudson.security.SecurityRealm jenkins.model.Jenkins.getSecurityRealm()`，同样也会调用到`NameBasedDispatcher#dispatch()`

![1568197833515](https://blog-1252261399.cos.ap-beijing.myqcloud.com/images/1568197833515.png)

接下来可以看到`ff.invoke()`返回一个`hudson.security.HudsonPrivateSecurityRealm`对象，然后重新调用`org.kohsuke.stapler.Stapler#invoke()`，这也是一个递归的过程。此时HudsonPrivateSecurityRealm返回的dispatchers有30个，在`Stapler#tryInvoke()`中进行循环调用，在每个dispatchers动态生成的dispatch方法中，会根据解析到的url路径与当前的dispatchers进行对比，不一致直接返回false，同时还会判断是否存在下一层路由，如果存在则进入doDispatch

比如此时解析到的url为/user/，则只有`hudson.security.HudsonPrivateSecurityRealm.getUser(String)`方法进入下一步doDispatch

![1568254883924](https://blog-1252261399.cos.ap-beijing.myqcloud.com/images/1568254883924.png)

当传入一个不存在的url，tryInvoke会返回false，抛出404，也就不继续往下解析了

![1568255416381](https://blog-1252261399.cos.ap-beijing.myqcloud.com/images/1568255416381.png)

经过上面递归的tryInvoke过程，Jenkins才完成路由解析，调用过程的流程图如下

![1568256409501](https://blog-1252261399.cos.ap-beijing.myqcloud.com/images/1568256409501.png)



## 动态路由绕过

- 公告：https://jenkins.io/security/advisory/2018-12-05/#SECURITY-595
- CVE：CVE-2018-1000861
- 影响版本：Jenkins<=2.153 / Jenkins LTS<=2.138.3

这是一个动态路由绕过导致未授权访问的问题，由Orange提交：）参考 [Hacking Jenkins Part 1 - Play with Dynamic Routing](https://devco.re/blog/2019/01/16/hacking-Jenkins-part1-play-with-dynamic-routing/)

### 白名单机制

上面分析了Jenkins构建动态路由的过程，主要调用的是`org.kohsuke.stapler.Stapler#tryInvoke()`方法，该方法对所属于`StaplerProxy`的类会有一次权限检查，而一开始我们知道除了`boundObjectTable`其他的node都被设置为`hudson.model.Hudson`，上面也讲到Hudson类继承自Jenkins，而Jenkins的父类`AbstractCIBase`是`StaplerProxy`的一个接口实现，所以除了`boundObjectTable`外所有node都会进行这个权限检查，具体实现在`jenkins.model.Jenkins#getTarget()`中

![1568259600209](https://blog-1252261399.cos.ap-beijing.myqcloud.com/images/1568259600209.png)

这个方法会先进行一次checkPermission，如果没有权限则会抛出异常还会再进行一次`isSubjectToMandatoryReadPermissionCheck`检查，如果这个检查通过同样会正常返回

![1568259863106](https://blog-1252261399.cos.ap-beijing.myqcloud.com/images/1568259863106.png)

这个检查中有一个白名单，如果存在于这个白名单中的url路由同样可以直接访问

![1568273111696](https://blog-1252261399.cos.ap-beijing.myqcloud.com/images/1568273111696.png)

```java
ALWAYS_READABLE_PATHS = ImmutableSet.of("/login", "/logout", "/accessDenied", "/adjuncts/", "/error", "/oops", new String[] {
    "/signup",
    "/tcpSlaveAgentListener",
    "/federatedLoginService/",
    "/securityRealm",
    "/instance-identity"
});
```

还是以`/securityRealm/user/admin/`为例，在解析至`securityRealm`的时候命中白名单，正常返回，而解析至`admin`的时候因为`User`类并非`StaplerProxy`子类，所以会跳过getTarget()检查，成功绕过

### 跨物件操作

接下来关注DescriptorByName

从继承关系图可以看到User也是`DescriptorByNameOwner`接口的实现

![1571821898844](https://blog-1252261399.cos.ap-beijing.myqcloud.com/images/1571821898844.png)

而`DescriptorByNameOwner`接口调用的是 `jenkins.model.Jenkins#getDescriptor`

```java
public interface DescriptorByNameOwner extends ModelObject {
    default Descriptor getDescriptorByName(String id) {
        return Jenkins.getInstance().getDescriptorByName(id);
    }
}
```

![1571822175710](https://blog-1252261399.cos.ap-beijing.myqcloud.com/images/1571822175710.png)

该方法首先获取了所有的descriptors，如果传入的id匹配到了相应的descriptor就能去调用指定的方法，例如`org.jenkinsci.plugins.scriptsecurity.sandbox.groovy.SecureGroovyScript#DescriptorImpl`，`getDisplayName()`和`doCheckScript()`都是能被调用到的

![1571823142780](https://blog-1252261399.cos.ap-beijing.myqcloud.com/images/1571823142780.png)

因此，通过构造`/securityRealm/user/DescriptorByName/xxx`的方式就可以调用到任意类的任意方法，只要满足下面两个条件：

1. 符合上文整理的命名规则；
2. 目标类继承了`Descriptor`；

利用链：

`Jenkins->HudsonPrivateSecurityRealm->User->DescriptorByNameOwner->Jenkins->Descriptor`

在这个漏洞修复后还想再利用则必须开启`Allow anonymous read access`匿名用户访问权限，否则会抛出404



## 总结

本报告上篇讨论了Jenkins动态路由机制和路由绕过的问题，通过这个脆弱点可以绕过用户权限检查从而访问到特定的物件，为下一步进行远程代码执行漏洞攻击降低了攻击门槛，是一个非常巧妙的入口。下篇将分析Jenkins主流插件Script Security中针对Groovy沙箱的绕过方法，欢迎关注。



参考

- <https://jenkins.io/security/advisories/>
- <https://devco.re/blog/2019/01/16/hacking-Jenkins-part1-play-with-dynamic-routing/>



<script>pangu.spacingPage();</script>