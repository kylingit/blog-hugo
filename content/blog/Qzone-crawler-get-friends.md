---
title: QQ 空间爬虫之获取好友
date: 2017-03-29 20:53:43
tags: [Python, Mysql, Qzone, crawler]
categories: Coding
---

<script src="https://ob5vt1k7f.qnssl.com/pangu.js"></script>

网上有些QQ空间爬虫都是首先设置访问权限为qq好友访问，然后获取所有好友信息。

其实QQ空间是有接口能够直接获取到所有好友的

### 获取好友
#### 普通信息
##### 接口地址
```
https://h5.qzone.qq.com/proxy/domain/r.qzone.qq.com/cgi-bin/tfriend/friend_show_qqfriends.cgi?uin=qq&fupdate=1&outCharset=utf-8&g_tk=g_tk
```
`g_tk`如何获取[上篇文章](https://kylingit.com/blog/qq-%E7%A9%BA%E9%97%B4%E7%88%AC%E8%99%AB%E4%B9%8B%E6%A8%A1%E6%8B%9F%E7%99%BB%E5%BD%95/)已经提过了

<!-- more -->

##### 数据形式
请求后返回的是json
```
_Callback(
{
code: 0,
subcode: 0,
message: "",
default: 0,
data: {
    items: [
        {
            uin: 12345,
            groupid: 2,
            name: "nick0",
            remark: "remark0",
            img: "http://qlogo4.store.qq.com/qzone/12345/12345/30",
            yellow: -1,
            online: 0,
            v6: 1
        },
        {
            uin: 23456,
            groupid: 8,
            name: "nick1",
            remark: "remark1",
            img: "http://qlogo3.store.qq.com/qzone/23456/23456/30",
            yellow: -1,
            online: 0,
            v6: 1
        },
        {
            uin: 34567,
            groupid: 1,
            name: "nick2",
            remark: "remark2",
            img: "http://qlogo4.store.qq.com/qzone/34567/34567/30",
            yellow: -1,
            online: 0,
            v6: 1
        }
    ],
  	gpnames: [
        {
            gpid: 0,
            gpname: "group0"
        },
        {
            gpid: 1,
            gpname: "group1"
        },
        {
            gpid: 2,
            gpname: "group2"
        }
    ]
}
```
关键的是data中的数据，除了所有好友的昵称备注头像外，还有所属的分组id等，本来可以根据这个`gpid`进行分组，可是找了一圈没找到如何显示所有分组信息的接口，于是这串数据就没派上用场了...

用一个session带上cookie请求这个接口就能获取所有好友了，可以先存下来，方便后面用。

#### 详细信息
可能有人认为这些信息还是太少了，既然抓取了就索性彻底一些，最好能获取到更详细的信息，于是又经过一番摸索，终于又get到一个接口：
##### 接口地址
```
https://h5.qzone.qq.com/proxy/domain/base.qzone.qq.com/cgi-bin/user/cgi_userinfo_get_all?uin=qq&fupdate=1&outCharset=utf-8&g_tk=g_tk
```
##### 数据形式
这是一个"详细版"的好友信息，包括空间名称，空间描述，出生年月，历史地理位置，现在地理位置等信息，以及更具体的邮箱，手机号等(如果有设置的话)
```
_Callback(
{
    code: 0,
    subcode: 0,
    message: "获取成功",
    default: 0,
    data: {
        uin: 12345
        is_famous: false,
        famous_custom_homepage: false,
        nickname: "nickname",
        emoji: [ ],
        spacename: "someone's qzone",
        desc: "",
        signature: "this is a signature",
        avatar: "http://b125.photo.store.qq.com/psb?/blabla",
        sex_type: 0,
        sex: 1,
        animalsign_type: 0,
        constellation_type: 0,
        constellation: 9,
        age_type: 0,
        age: 18,
        islunar: 0,
        birthday_type: 0,
        birthyear: 1999,
        birthday: "01-01",
        bloodtype: 0,
        address_type: 0,
        country: "中国",
        province: "",
        city: "北京",
        home_type: 0,
        hco: "中国",
        hp: "北京",
        hc: "东城",
        marriage: 0,
        career: "",
        company: "",
        cco: "",
        cp: "",
        cc: "",
        cb: "",
        mailname: "",
        mailcellphone: "",
        mailaddr: "",
        qzworkexp: [ ],
        qzeduexp: [ ],
        ptimestamp: 1450773545
    }
}
)
```
嗯...只要获取到每个好友的qq后接着请求这个接口，更详细的信息就得到了～乖乖存下来

### 数据库设计
对了，本爬虫是基于Python和MySQL的，所以数据都会存在MySQL数据库中，设计为每个好友一个库，含有说说表，说说评论表，说说点赞表，留言表，留言回复表等。
首先好友信息只要获取一遍，存在登录qq的好友表中，字段都是上面获取的数据
```
create_tb_sql = 'CREATE TABLE IF NOT EXISTS %s\
    (id int primary key, \
    uin varchar(15), \
    sex int(2), \
    groupid int(2), \
    nickname varchar(40), \
    remark varchar(20), \
    spacename varchar(50), \
    age int(2), \
    birthday varchar(20), \
    city varchar(20), \
    img varchar(60), \
    yellow int(2), \
    online int(2), \
    v6 int(2)) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4' % tablename		#change mysql encoding to support emoji
```
这里建表最后的`ENGINE=InnoDB DEFAULT CHARSET=utf8mb4`需要解释一下，因为有好多好友的昵称，签名等都是含有emoji表情的，emoji虽然也有编码，但它是用4字节来存储的，而 MySQL 中 utf8 的字段只能存储 1 至 3 字节的字符，所有直接存储会出错，这里就在建表的时候设置表的编码格式为`utf8mb4`，该编码是`utf8`的超集，向下兼容`utf8`，可以参考前阵子写的文章 [PYTHON 使用 MYSQL 存储 EMOJI 表情](https://kylingit.com/blog/python-%E4%BD%BF%E7%94%A8-mysql-%E5%AD%98%E5%82%A8-emoji-%E8%A1%A8%E6%83%85/)

字段含义就不用解释了，注意一下的是`birthday`字段是拼接出生年份和具体月日的，就不细分了，`city`字段拼接国家省份和城市。`sex`字段为`0`的表示无法获取该好友的信息

预览

![qq friends](https://ob5vt1k7f.qnssl.com/DNYkv)

### 结束语
看似普通的get访问，用request方便又轻松，实际上背后有很多坑...比如说有些上个年代遗留的火星文...又比如说各种有意无意在签名中啊说说中啊等带各种"特殊字符"的，不做过滤直接让程序逼停...

<script>pangu.spacingPage();</script>