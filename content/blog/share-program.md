---
title: 分享几个有用的小程序
date: 2017-03-18 23:21:02
tags: Linux
categories: Coding
---
<script src="https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/pangu.js"></script>

### 1. [一個改良的 Ping](http://fangpeishi.com/improved_ping.html)
有时候想 ping 一个网址，直接从浏览器复制会带上`http://`，粘进命令行就出错了...
于是可以用这个脚本代替 ping 程序，改成`pin`放在`/bin`下就 OK 了
```
#!/usr/bin/env bash
#author: fangpeishi@gmail.com
#issues:
#  - http(s)://xxx.xx/xxx/xx?xxx
#  - 192.168.1.1/32

new_args=`echo $@ |sed  's/http.*\:\/\///' |sed 's/\/[^ ]*//'`
#echo ${new_args}
ping -c 4 ${new_args}
```
[作者](http://fangpeishi.com)还带上另一个功能，一个网段的也能用
`ping 192.168.1.1/32` ----> `ping 192.168.1.1`
![Untitled Image](https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/images/QSTrD)

<!-- more -->

### 2. 随机密码生成脚本
可以自定义长度
```
#!/bin/bash
L=$2
if [ ! -z $1 ];then
  if [[ "$1" =~ ^[0-9]+$ ]];then
    L=$1
  fi
fi
</dev/urandom tr -dc '0123456789abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ~!@#$%^&*()_+' | head -c${L}; echo ""
```
![Untitled Image](https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/images/ooNjB)

### 3. 不定期更新

<script>pangu.spacingPage();</script>