---
title: 关于 MHN 的一些部署介绍
date: 2016-09-30 14:32:18
tags: [MHN,Honeypot]
categories: Coding
---
<script src="https://ob5vt1k7f.qnssl.com/pangu.js"></script>

### 简介
[MHN (Modern Honey Network)](https://itandsecuritystuffs.wordpress.com/2015/02/03/honeypot-networks/) 是一个开源软件，它集成了多种蜜罐，简化了蜜罐的部署，同时便于收集和统计蜜罐的数据。
它包括

- Sort: https://www.snort.org/

- Suricata: http://suricata-ids.org/

- Dionaea: http://dionaea.carnivore.it/, 它是一个低交互式的蜜罐，能够模拟MSSQL, SIP, HTTP, FTP, TFTP等服务 drops中有一篇介绍： http://drops.wooyun.org/papers/4584

- Conpot: http://conpot.org/

- Kippo: https://github.com/desaster/kippo, 它是一个中等交互的蜜罐，能够下载任意文件。 drops中有一篇介绍： http://drops.wooyun.org/papers/4578

- Amun: http://amunhoney.sourceforge.net/, 它是一个低交互式蜜罐，但是已经从2012年之后不在维护了。

- Glastopf： http://glastopf.org/

- Wordpot： https://github.com/gbrindisi/wordpot

- ShockPot： https://github.com/threatstream/shockpot, 模拟的CVE-2014-6271，即破壳漏洞

- p0f： https://github.com/p0f/p0f

具体的介绍可以看 WooYun Drops 的一篇文章 [蜜罐网络](https://ob5vt1k7f.qnssl.com/%E8%9C%9C%E7%BD%90%E7%BD%91%E7%BB%9C.html)

安装过程不再赘述，讲讲实际部署中遇到的一些问题

<!-- more -->

### 一些注意点
#### 一
首先，MHN honeymap 是一个控制端，可以认为是“总部”，把它安装在一台机器上就好，它负责记录和显示所有子节点上的攻击数据，部署脚本也在这台机器上，想要部署一个蜜罐到多台服务器上，只要运行它提供的部署脚本就可以，安装和配置都会自动完成，结果会显示在这台控制端。

先上一个正常运行的页面

![HoneyMap](https://ob5vt1k7f.qnssl.com/HoneyMap.jpg)

因为部署了差不多100台蜜罐分布在全球不同的地区，这里看上去还是蛮壮观的。红色的代表正在发起攻击的 ip 地理位置，黄色的代表受攻击方，也就是蜜罐的 ip ，显示结果是实时的。

这是统计信息，包括 TOP 5 IPs, TOP 5 ports, TOP 5 Honey Pots, TOP 5 Sensors, TOP 5 Attacks Signatures，所有记录都会保存在 Mongodb 数据库中，这里只是显示过去 24 小时的

![Dashboard](https://ob5vt1k7f.qnssl.com/Dashboard.jpg)

#### 二
MHN 的 HoneyMap 显示页面使用的是 3000 端口，控制页面运行在 80 端口，80 端口需要验证登录，3000 端口是开放的，建议用 iptables 对端口进行一些保护，需要注意的是，当利用脚本部署不同的蜜罐时，需要从这台服务器下载脚本和配置文件，这时候 80 端口不能被设置成不允许访问，不然没法部署成功。

#### 三
Map 页面就是显示的地图，这里一开始点击会显示说 404，需要在配置文件把页面地址更改一下
`vim /opt/mhn/server/config.py`

把
`HONEYMAP_URL = ':3000' `

改成
`HONEYMAP_URL = 'http://ip:3000'`   
 
 ip 即为该服务器 ip
 
![Map](https://ob5vt1k7f.qnssl.com/map.jpg)

Deploy 页面即保存部署蜜罐的脚本，可以下拉选择不同的蜜罐，也可以自定义脚本。它提供的脚本不一定完全适用，有些情况需要自己更改一下。复制上面的 Deploy Command 到另外的机器上执行，完成后一般就会在 Sensors 显示新安装的蜜罐节点。

![Deploy](https://ob5vt1k7f.qnssl.com/Deploy.jpg)

Attacks 页面显示的是具体的攻击情况，包括时间，节点，国家，源ip和源端口，使用的协议以及发生在哪个蜜罐上

![Attacks](https://ob5vt1k7f.qnssl.com/Attacks.jpg)

Plyloads 页面显示一些攻击行为的 payload，包括 sql 注入，恶意扫描等，它会记录所有 url 访问

Rules 页面可以自定义一些触发规则，一般保持默认就好

Sensors 是各个蜜罐节点的情况，成功运行的蜜罐会在这里显示

Charts 页面会显示 ssh 蜜罐收集到的用户名密码等，这里显示的是 cowrie 捕捉到的一些 ssh 尝试登录，以及 top 用户名和密码

![Charts](https://ob5vt1k7f.qnssl.com/passwd.jpg)


### 关于具体的蜜罐

#### cowrie
这是一个中等交互的SSH蜜罐，关于它的介绍在 Freebuf 上有一篇 [文章](http://www.freebuf.com/articles/network/112065.html)，如何安装如何配置在上面都有，不再赘述。
如果运行出错，查看日志发现问题 
> getDSAkeys - TypeError: must be long, not mpz 

这是由于 Twisted 库不兼容引起的，解决办法是
```
$ cd cowrie/data
$ ssh-keygen -t dsa -b 1024 -f ssh_host_dsa_key
```
需要注意尽量不要将 22 端口暴露在公网，可以使用 iptables 转发到 cowrie 的运行端口，同时将正常的 ssh 服务运行在另外的端口。


#### dionaea
这是一个低交互式的蜜罐，能够模拟MSSQL, SIP, HTTP, FTP, TFTP等服务，安装过程参考上面的文章
如果运行 
> supervisorctl status 

遇到问题
```
unix:///var/run/supervisor.sock refused connection
```
解决办法是
```
sudo supervisord -c /etc/supervisor/supervisord.conf
sudo supervisorctl -c /etc/supervisor/supervisord.conf
```

#### glastopf
glastopf 是一款低交互式 Web 蜜罐，它能记录包括 sql 注入，XSS 攻击等常见的 Web 攻击类型

![Payload](https://ob5vt1k7f.qnssl.com/Payloads.jpg)

#### suricata
网络入侵检测和阻止引擎
这个蜜罐没有运行成功，后续解决掉问题可能会更新上来。

目前部署的就这几个蜜罐，其它的可能会根据需要安装部署，有时间的话会更新。

<script>pangu.spacingPage();</script>



