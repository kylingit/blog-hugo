---
title: sqlite 执行删除操作后文件大小不变的解决办法
date: 2016-08-27 13:13:18
tags: Python
categories: Coding
---
<script src="https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/pangu.js"></script>

在用python对sqlite3数据库进行删除部分数据的操作后，数据库文件大小并没有改变，上网找了找原因，发现确实是这样 :

> When an object (table, index, trigger, or view) is dropped from the database, it leaves behind empty space. This empty space will be reused the next time new information is added to the database. But in the meantime, the database file might be larger than strictly necessary. Also, frequent inserts, updates, and deletes can cause the information in the database to become fragmented - scrattered out all across the database file rather than clustered together in one place.

当一个对象（表，索引，触发器或视图）被从数据库中删除，留下一块空白空间。这块空间将被下一次新的信息添加到数据库中重复使用。但在此期间，数据库文件可能变得非常大。此外，频繁的插入，更新和删除可能会导在数据库中的信息成为零散的碎片分布在数据库中，而不是在一个地方聚集在一起。

解决办法是

> 1. 在数据删除后，手动执行 "VACUUM" 命令
> 2. 在数据库文件创建时，将 auto_vacuum 设置成 "1" 。

<!-- more -->

但是第二个方法有一定的限制，它只会从数据库文件中截断空闲列表中的页，而不会回收数据库中的碎片，也不会像`VACUUM` 命令那样重新整理数据库内容。实际上，由于需要在数据库文件中移动页， `auto-vacuum` 会产生更多的碎片。而且，在执行删除操作的时候，会产生一个`.db-journal`文件。 
使用 `auto-vacuum` 的前提是，数据库中需要存储一些额外的信息以记录它所跟踪的每个数据库页都能找回其指针位置。 所以，`auto-vacumm` 必须在建表之前就开启。在一个表创建之后， 就不能再开启或关闭 `auto-vacumm`。

在python中就执行
```
import sqlite3
conn = sqlite3.connect(dbfile)
sql = 'delete from table where ...'
cu = conn.cursor()
cu.execute(sql)
cu.execute('vacuum')
cu.close()
conn.close()
```
<script>pangu.spacingPage();</script>




