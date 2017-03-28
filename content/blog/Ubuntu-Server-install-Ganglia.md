---
title: Ubuntu Server 安装配置 Ganglia
date: 2016-07-29 17:54:29
tags: Ganglia
categories: Coding
---
<script src="https://ob5vt1k7f.qnssl.com/pangu.js"></script>

Ganglia是一个开源的集群监控系统，它基于分层设计，使用广泛的技术，如XML数据代表，便携数据传输，RRDtool用于数据存储和可视化，可以方便地对机器性能及系统运行状态进行可视化监控，广泛应用于对Hadoop，Spark等分布式系统的监测管理。

### 简介
Ganglia的核心包含gmond、gmetad以及一个Web接界面。主要是用来监控系统性能，如：cpu 、mem、硬盘利用率， I/O负载、网络流量情况等，通过曲线很容易见到每个节点的工作状态，对合理调整、分配系统资源，提高系统整体性能起到重要作用。

<!-- more -->

![Ganglia][1]
#### Gmond
Ganglia monitoring，它是一个守护进程，用于收集机器内的metric，它还可以接受别的node发送过来的metric，并且保存一小段时间（几十秒），运行在每一个需要监测的节点上，收集监测统计，发送和接受在同一个组播或单播通道上的统计信息。Gmond可以扮演下面三种角色： 

- 收集metric并发送出去，同时也接收别的node发送过来的metric；
- 只采集metric并发送出去（关键字 deaf）；
- 只接收别的机器发送过来的metric（关键字 mute）；
默认情况下，gmond监听8649端口，用来发送和接收udp，tcp数据包。

#### Gmetad
Ganglia meta daemon，也是一个守护进程，定期检查gmonds ，从那里拉取数据，并将他们的指标存储在RRD存储引擎中。它可以查询多个集群并聚合指标。默认情况下gmond通过multicast的方式发送自己采集到的数据，整个Multicast group里的node都拥有整个cluster的全部metrics。而gmetad可以从一个cluster的任意一个node拿到整个cluster的全部metric并记录到rrd数据库。
默认情况下，gmetad监听8651端口，从这里可以拿到gmetad存放的最新metric数据，也可以给更高层的gmetad使用；监听8652端口，提供数据查询接口，供web使用。

#### RRD
运行在主节点的一个工具，轮询调度数据库，用于存储数据和可视化时间序列。RRD也被用于生成用户界面的web前端。


### 安装
#### 安装环境
Ubuntu Server 12.04 (10.24.84.23    master node)
Ubuntu Server 16.04 (10.24.84.24    client node)
均需要安装apache服务

#### 主节点 master node
```
sudo apt-get install ganglia-monitor 
sudo apt-get install rrdtool 
sudo apt-get install gmetad 
sudo apt-get install ganglia-webfrontend
```
#### 子节点 client node
```
sudo apt-get install -y ganglia-monitor
```

### 配置
#### 主节点 master node
##### 修改 gmetad 配置
`vim /etc/ganglia/gmetad.conf`
```
# data_source,cluster的名字,可以自定义，但要与gmond配置对应起来
data_source "my cluster" localhost
```
##### 修改 gmetad 配置
`vim /etc/ganglia/gmond.conf`
```
# 修改 cluster name
cluster {
  name = "my cluster" ## use the name from gmetad.conf
  owner = "unspecified"
  latlong = "unspecified"
  url = "unspecified"
}
# 修改 udp_send_channel，添加 host
udp_send_channel = {
  # mcast_join = xxx.xxx.xxx.xxx
  host = localhost
  port = 8649
  ttl = 1
}
# 修改 udp_recv_channel，注释掉 mcast_join, bind
udp_recv_channel = {
  # mcast_join = xxx.xxx.xxx.xxx
  port = 8649
  # bind = xxx.xxx.xxx.xx
}
```
##### 重启 ganglia-monitor 和 gmetad 服务
```
service gmetad restart
service ganglia-monitor restart
```

#### 子节点 client node
##### 修改 gmetad 配置 
```
# 修改 cluster name
cluster {
  name = "my cluster"
  owner = "unspecified"
  latlong = "unspecified"
  url = "unspecified"
}
# 修改 udp_send_channel host
udp_send_channel {
  # mcast_join = xxx.xxx.xxx.xxx
  host = master node ip
  port = 8649
  ttl = 1
}
udp_recv_channel {
  # mcast_join = xxx.xxx.xxx.xxx
  port = 8649
  # bind = xxx.xxx.xxx.xx
}
```
##### 重启 ganglia-monitor 服务
`service ganglia-monitor restart`

#### 配置apche2
##### 复制到apache www目录
`sudo cp -r /usr/share/ganglia-webfrontend /var/www/ganglia`
##### 重启apache服务
`sudo /etc/init.d/apache2 restart`


### 使用
直接访问`http://10.24.84.23/ganglia`就能看到ganglia成功运行

![ganglia_master][2]


  [1]: https://ob5vt1k7f.qnssl.com/ganglia.png
  [2]: https://ob5vt1k7f.qnssl.com/ganglia_master.png

<script>pangu.spacingPage();</script>