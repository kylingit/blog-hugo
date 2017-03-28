---
title: 对吉林大学某站的一次渗透测试
date: 2016-06-14 22:00:18
tags: Wooyun
categories: Hacking
---
<script src="https://ob5vt1k7f.qnssl.com/pangu.js"></script>

### 简要描述

后台验证码机制不完善——爆破弱口令——后台突破上传——内网渗透

<!-- more -->

### 详细说明

吉林大学继续教育学院&网络教育学院

`http://202.198.17.226:8088/oa/index.php`

![Untitled Image](https://ob5vt1k7f.qnssl.com/0.jpg)

登录失败时没有刷新验证码，所以验证码机制形同虚设，上burp爆破一下，发现弱口令

```
admin
000000
```
![Untitled Image](https://ob5vt1k7f.qnssl.com/01.jpg)

找到上传，添加可上传的后缀，getshell

![Untitled Image](https://ob5vt1k7f.qnssl.com/02.jpg)

![Untitled Image](https://ob5vt1k7f.qnssl.com/03.jpg)

翻看到了数据库配置

![Untitled Image](https://ob5vt1k7f.qnssl.com/4.jpg)

连接成功，然后发现了不少用户，还用相同的弱口令...

![Untitled Image](https://ob5vt1k7f.qnssl.com/3.jpg)

![Untitled Image](https://ob5vt1k7f.qnssl.com/2.jpg)

上传GetPass跑了一下密码，居然跑出了不少...字母+数字+特殊字符，还好没爆破= =...

![Untitled Image](https://ob5vt1k7f.qnssl.com/6.jpg)

3389登录成功

![Untitled Image](https://ob5vt1k7f.qnssl.com/7.jpg)

执行net view发现不显示，但确实在内网，一时无法下手，忽然看到了CuteFTP，想到能不能用ftp得到些信息呢，于是上了获取密码的工具，得到一些密码

![Untitled Image](https://ob5vt1k7f.qnssl.com/04.jpg)

登录试试

![Untitled Image](https://ob5vt1k7f.qnssl.com/12.jpg)

现在已经有了一些密码，再看了一下mstsc的记录，猜测一下网络结构，应该有至少三台服务器，226,227和228，并且开启了ftp服务。同时应该有229,231也运行着ftp，猜测一下应该也是三台服务器(加上230)，但是运行着linux操作系统，后来打开桌面上的SSH Secure Shell Client,发现确实三台linux，ftp同样可以登录，还有两台忘记截图了...

![Untitled Image](https://ob5vt1k7f.qnssl.com/13.jpg)

这时候看到几个ip不属于这个网段，但是互相可以连接，而且外网访问会指向同一个站点，于是就想难道是分配了多个独立ip? 好有钱...

然后看到了这个...

![Untitled Image](https://ob5vt1k7f.qnssl.com/16.jpg)

顿时一目了然了，和猜测的能吻合一部分，3台DEC服务器应该放着站点，3台Web服务器，2台数据库服务器，每台服务器都有三个ip，当时的心理活动是= = 一开始被搞得晕头转向...下面192.168.3.1是网关，administrator能登录...站库分离，值得提倡，却因为一处验证码不完善以及弱口令彻底沦陷...

继续渗透，通过ftp传了一个sql回来，发现大量用户信息...和更多的口令，最终终于集齐了三台服务器...

![Untitled Image](https://ob5vt1k7f.qnssl.com/18.jpg)

![Untitled Image](https://ob5vt1k7f.qnssl.com/14.jpg)

继续渗透，发现三台linux有两台是可以用root登录的，还有一台大概密码过期了，上上张图就能看到...

本来还想找找数据库里到底放着什么→_→ 但是时间原因就不继续了...

当看到一套完善的系统和众多的密码认证，会有一种不可抗拒的吸引力...真得到了反而不知道该干什么了...

### 修复方案

- 完善验证码机制

- 修改弱口令

- 软件的保存会话功能会暴露不少信息，除了用脑袋记住暂时没想到什么有效的方法能方便又省力得进行管理...

### 乌云

> [对吉林大学某站的一次渗透测试](http://wooyun.org/bugs/wooyun-2016-0214638)

<script>pangu.spacingPage();</script>