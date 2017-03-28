---
title: salt-ssh 配置使用
date: 2016-09-15 14:54:29
tags: [Python,SSH,Linux]
categories: Coding
---
<script src="https://ob5vt1k7f.qnssl.com/pangu.js"></script>

salt-ssh 是 Saltstack 框架下的一款批量化远程操作工具，具体介绍可以看 [这里](http://opensgalaxy.com/2015/08/13/saltstack%E5%85%A5%E9%97%A8%E3%80%90salt-ssh%E3%80%91%E4%BD%BF%E7%94%A8/) 
关于 Saltstack，它是一款自动化运维工具，具体可以浏览 [官网](https://saltstack.com)，这里只介绍一下 salt-ssh 的使用。

<!-- more -->

salt-ssh 的配置很简单，在 /etc/salt/ 下修改 roster 文件，把需要管理的服务器 ip，用户名，密码按格式配置好即可
> vim /etc/salt/roster

```
server00:
 host: x.x.x.x
 user: root
 passwd: root
 
server01:
 host: x.x.x.x
 user: root
 passwd: root
```
 然后测试一下能不能连通就好了
> salt-ssh '*' test.ping

'*' 是指所有节点，想要单独某个节点的话指定就可以了
> salt-ssh server00 test.ping
 
 可能需要验证是否接受密钥，不想被提示就加上参数 -i
> salt-ssh '*' test.ping -i

测试能够连通就可以执行命令了，使用参数 -r
> salt-ssh '*' -r 'uname -a' -i

这里要说的是配置文件里明文记录密码是十分不安全的行为，极端情况是某台服务器被入侵，发现了这个文件，恰巧又有大量服务器配置在这，相当于把机器送到黑客手上了。
即使是加密后的密码也不安全，总之是用文件记录敏感信息都是不负责任的做法。

想要不在配置文件中记录密码，可以在执行命令的时候把密码作为参数
> salt-ssh '*' --passwd 'password' -r 'args' -i

而配置文件里只要记录 ip 和用户名就可以
```
server00:
 host: x.x.x.x
 user: root
 
server01:
 host: x.x.x.x
 user: root
```
这样做的优点是不会在文件中泄露密码，缺点是假如每台机器密码不一样，执行起来会比较麻烦，各自取舍吧。 也有通过 keys 验证身份，但测试之后发现还是得认证身份，这里就不提了。

Tips
其实直接在命令中指定密码依然十分危险，因为命令记录会把你出卖...可以执行一下
> cat ~/.bash_history

所以涉及到输入密码的命令，可以在输入前键入一个空格，即按一下空格再正常输入命令，这样这条命令就不会被记录在历史里。

<script>pangu.spacingPage();</script>