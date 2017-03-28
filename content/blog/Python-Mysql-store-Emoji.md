---
title: Python 使用 Mysql 存储 Emoji 表情
date: 2017-02-20 19:43:18
tags: [Python, Mysql]
categories: Coding
---
<script src="https://ob5vt1k7f.qnssl.com/pangu.js"></script>

最近使用 Python 处理数据的时候遇到 mysql 存储 emoji 表情的问题，觉得可以总结一下。

#### 一. 报错信息
```
Incorrect string value: '\xF0\x9F\x91\x8D' for column 'xxx'
```

#### 二. 错误分析
从异常能看出这是编码的问题，当前的配置是数据库连接使用 utf8，字符集也是 utf-8。查阅资料发现，在 mysql 中 utf8 的字段只能存储 1 至 3 字节的字符，而 emoji 表情是使用 4 字节字符来表示的，这就导致 `Incorrect string value` 错误。

#### 三. 解决办法
#####  1. 使用 `utf8mb4` 编码存储数据，`utf8mb4 is a superset of utf8`
utf8mb4 向下兼容 utf8，在 Mysql 5.5.3 以上版本支持 utf8mb4


##### 方法(1)
修改 mysql 配置. 编辑 my.ini 文件，之后要重启 mysql 服务
```
[client]
default-character-set = utf8mb4		# 客户端来源数据的默认字符集

[mysqld]
character-set-server = utf8mb4		# 服务端默认字符集
collation-server = utf8mb4_unicode_ci	# 连接层默认字符集

[mysql]
default-character-set = utf8mb4		# 数据库默认字符集

```
##### 方法(2)
<!-- more  -->
在 python 连接数据库和创建表时指定编码
```
import MySQLdb
# 连接
conn = MySQLdb.connect("127.0.0.1", "user", "passwd")
cursor = self.conn.cursor()
cursor.execute("SET NAMES utf8mb4")
cursor.execute("SET CHARACTER SET utf8mb4")
cursor.execute("SET character_set_connection = utf8mb4")

# 建库
cursor.execute('CREATE DATABASE IF NOT EXISTS %s CHARACTER SET utf8mb4 \ 
	COLLATE utf8mb4_unicode_ci' % dbname)

# 建表
cursor.execute('CREATE TABLE table(id int primary key, name char(10))') \
	ENGINE = InnoDB DEFAULT CHARSET = utf8mb4
```
可以查询 mysql 编码方式
` show variables like 'character_set_%';`
![mysql_charset](https://ob5vt1k7f.qnssl.com/mysql_charset.png)

#### 2. 使用正则表达式过滤 emoji 字符
```
#!/usr/bin/env python
import re

text = u'This dog \U0001f602'
print(text) # with emoji

emoji_pattern = re.compile("["
        u"\U0001F600-\U0001F64F"  # emoticons
        u"\U0001F300-\U0001F5FF"  # symbols & pictographs
        u"\U0001F680-\U0001F6FF"  # transport & map symbols
        u"\U0001F1E0-\U0001F1FF"  # flags (iOS)
                           "]+", flags=re.UNICODE)
print(emoji_pattern.sub(r'', text)) # no emoji
```

<script>pangu.spacingPage();</script>