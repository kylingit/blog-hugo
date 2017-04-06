---
title: QQ 空间爬虫之爬取留言
date: 2017-04-01 19:50:04
tags: [Python, Mysql, Qzone, crawler]
categories: Coding
---

<script src="https://ob5vt1k7f.qnssl.com/pangu.js"></script>

今天来讲爬取所有留言吧~

### 接口分析
惯例，从url接口入手

我们分析一个请求首先要抓到与服务器交互的数据包，这就要用到抓包工具，像`Burpsuite`,`Fiddler`等，为了方便有时候也直接用`chrome`的审查元素

登录空间，打开`f12`，切换到`network`选项卡，然后点击留言板，注意下面的请求，找到获取留言的链接
![http request](https://ob5vt1k7f.qnssl.com/xNpNZ)

它长这样

```
https://h5.qzone.qq.com/proxy/domain/m.qzone.qq.com/cgi-bin/new/get_msgb?uin=登录qq&hostUin=目标qq&start=起始位置&num=一次获取数量&format=jsonp&inCharset=utf-8&outCharset=utf-8&g_tk=g_tk值
```

关键参数是：`uin`,`hostUin`,`start`,`num`,`g_tk`,分别对应`登录qq`，`目标qq`，`起始位置`，`一次性获取的留言条数`，`g_tk值`，构造出这些参数，就能获取一个好友的所有留言了

值得注意的是`start`和`num`，由于该接口的限制，一次最多只能获取20条留言，也就是说`num`值最大为20，这就无法一次性获取所有的留言，需要按`20条每份`切割开来，每获取20条就让`start`增加20，留言总数可以在返回的数据中获取到，然后注意控制边界，我们就能`一份一份`获取所有数据了

<!-- more -->

### 数据分析
右键新标签页打开这个链接，分析返回的数据

![http reply](https://ob5vt1k7f.qnssl.com/PFpqZ)

这也是一串json格式的数据，这样我们就能看到一条留言的存储结构，包括留言总数，留言者，留言内容，留言时间，回复内容等等，所有都get到了，然后设计数据库，存下来就ok了

可是真的这么顺利吗...

### 踩坑
#### 坑一
当一切准备就绪，摩拳擦掌准备大干一场的时候，忽然发现，对方设置了访问权限...

![cannot access](https://ob5vt1k7f.qnssl.com/IqCvJ)

![sad](https://ob5vt1k7f.qnssl.com/ljsdK)

好吧，再正常不过的事了，谁还没有个小秘密呢(~~人家根本就不想让你看好嘛~~)

不能看到留言就不能抓到数据了，那如何判断对方是不是不让你访问ta的空间呢？可以看到，当对方设置访问权限的时候，返回的状态码是不一样的，我们可以根据这个状态码`code`来判断

但是再想一下，如果我们要获取说说数据，碰到同样的情况，难道也是来先请求一次留言的接口？也不是不可以，但最好把这两者独立开，避免不同内容混杂在一起。也有可能获取说说的时候又有不一样的状态码，那到时候再判断行不行呢？当然也是可以的...

呃...其实关键的是我们最好找到一个通用的接口，根据这个接口返回的状态做一次判断，这样就能在所有子模块中决定是否对这个好友继续爬取数据，那这个接口是什么呢？

在上次[获取好友信息](https://kylingit.com/blog/qq-%E7%A9%BA%E9%97%B4%E7%88%AC%E8%99%AB%E4%B9%8B%E8%8E%B7%E5%8F%96%E5%A5%BD%E5%8F%8B/)的那部分中，有一步是根据qq获取[详细信息](https://kylingit.com/blog/qq-%E7%A9%BA%E9%97%B4%E7%88%AC%E8%99%AB%E4%B9%8B%E8%8E%B7%E5%8F%96%E5%A5%BD%E5%8F%8B/#详细信息)，这个详细信息的获取是有好友权限的，不然就可以得到任意qq的信息了...扯远了，好友权限意味着你必须可以访问ta的空间，这对于设置了访问限制的好友也是一样的，我们同样无法获取到被限制访问的好友的具体信息，于是我们可以再次利用这个接口

```
https://h5.qzone.qq.com/proxy/domain/base.qzone.qq.com/cgi-bin/user/cgi_userinfo_get_all?uin=qq&fupdate=1&outCharset=utf-8&g_tk=g_tk
```

当对方限制了我们的访问权限，同样返回一个`-4009`状态码，还有`您无权访问`的提示信息，这两个都可以用来判断对方是否对我们设置了访问权限

![cannot access](https://ob5vt1k7f.qnssl.com/pfccO)

#### 坑二
好了，现在我们解决了没有访问权限的问题，抛弃了那些早已抛弃我们的小伙伴，再次兴致勃勃地准备大干一场(雾)的时候，忽然发现，私密留言...

![secret message](https://ob5vt1k7f.qnssl.com/y81Qe)
![so sad](https://ob5vt1k7f.qnssl.com/3I4Aj)

好吧，再正常不过的事了，谁还没有好几个小秘密呢(~~人家双方都不想让你看好嘛~~)

私密留言是看不到具体内容的，一味地取`Content`的内容肯定是会出错的，所以还是提前加个判断，判断`secret`的值就好了，很简单

好吧，其实这也不能算是坑了，都是设计过程中要注意的地方，把所有情况都要考虑到。

### 数据库设计
接下来设计数据库，还是为每个好友建立一个独立的表，暂且叫做`qq_messages`吧

留言信息跟好友信息不一样，因为它还有回复，回复也有自己的内容，时间，回复者等信息，所以有一个层次关系，回复的内容嵌在留言内容下面，类似树的结构，所以靠一张表不能很好地表示整个留言关系，于是设计再建一张表，叫做`qq_messages_reply`，专门存放一条留言下的回复信息，和留言表有一个key对应，也就是每条留言独有的`msgid`，可以认为是外键吧，但这里没有设置成外键，因为感觉不需要...

- `qq_messages`结构

```
create_tb_sql = 'CREATE TABLE IF NOT EXISTS %s\
    (id int primary key, \
    msgid varchar(15), \			#留言的唯一标识
    uin varchar(15), \
    nickname varchar(50), \
    secret int(2), \				#私密留言标识
    bmp varchar(20), \
    pubtime varchar(20), \
    modifytime varchar(20), \
    effect char(10), \				#下面三个字段不清楚做什么的，但还是留着吧
    type int(2), \
    capacity varchar(10), \
    ubbContent TEXT, \				#留言内容，注意TEXT
    replyFlag int(2))' % tablename		#是否有回复
```
注意：由于无法确定留言内容的长度，所以不能确定用多大的存储空间来存储，所以这里将存储结构设置成`TEXT`，`TEXT`的存储空间是`65 535`个字节，大约可以存储`20000`个汉字

关于MySQL可以存储的数据类型以及存储空间，可以参考[文档](https://www.runoob.com/mysql/mysql-data-types.html)

可以注意到在`qq_messages`中多了一个`replyFlag`字段，这个字段是自己加的，用来区分该留言有没有回复，这是根据`replyList`是否为空来判断的

预览

![qq messages](https://ob5vt1k7f.qnssl.com/cIIY1)

- `qq_messages_reply`结构

```
create_tb_sql = 'CREATE TABLE IF NOT EXISTS %s\
    (id int primary key, \
    msgid varchar(15), \
    replycount char(4), \
    uin varchar(15), \
    nickname varchar(50), \
    pubtime varchar(20), \
    content TEXT)' % tablename
```
回复表跟上面差不多，就不解释了

预览

![qq message reply](https://ob5vt1k7f.qnssl.com/5pPWS)

插入数据的时候根据回复数插入空值，使看上去有层次关系

### 结束语
剩下的就是一些小细节了，比如说私密留言获取不到留言者的`uin`以及具体的内容，而表的字段已经固定了，无法正确插入怎么办呢？

留言的内容含有一些特殊字符，比如`\`，`'`等，让sql语句被转义或被截断，又该怎么办呢？

还有，留言的回复中又有对话，而且有很多条，这个情况又怎么处理呢？

╮(╯_╰)╭


<script>pangu.spacingPage();</script>