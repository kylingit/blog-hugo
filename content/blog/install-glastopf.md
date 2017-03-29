---
title: Ubuntu 16.04 安装 Glastopf
date: 2016-07-22 14:32:18
tags: Honeypot
categories: Coding
---
<script src="https://ob5vt1k7f.qnssl.com/pangu.js"></script>

`glastopf`是一个开源的低交互式Web应用蜜罐，用python编写，可以在各种操作系统上部署，可以方便监控和捕获恶意文件样本，攻击方式等，后期也有开源脚本利于统计分析，同时方便安装配置。 
 
### 项目地址
> https://github.com/mushorg/glastopf 

### 安装依赖
python要求2.7+，pip要求2.7+

<!-- more -->

#### pymongo
`pip2.7 install --upgrade pymongo`
#### numpy and other deps
```
pip2.7 install numpy
pip2.7 install chardet sqlalchemy lxml beautifulsoup pyOpenSSL requests MySQL-python
pip2.7 install scipy 
```
(be warned: pip installs software from alpha centauri so expect *some* delays. also compiling can take a while.)
#### antlr
```
wget http://www.antlr3.org/download/antlr-3.1.3.tar.gz
tar xzf antlr-3.1.3.tar.gz
cd antlr-3.1.3/runtime/Python
python2.7 setup.py install
```
#### SKLearn
```
git clone git://github.com/scikit-learn/scikit-learn.git
cd scikit-learn
python2.7 setup.py install
```
#### evnet
```
git clone git://github.com/rep/evnet.git
cd evnet
python2.7 setup.py install
```

### 或者直接安装
```
sudo apt-get update
sudo apt-get install python2.7 python-openssl python-gevent libevent-dev python2.7-dev build-essential make
sudo apt-get install python-chardet python-requests python-sqlalchemy python-lxml
sudo apt-get install python-beautifulsoup mongodb python-pip python-dev python-setuptools
sudo apt-get install g++ git php5 php5-dev liblapack-dev gfortran libmysqlclient-dev
sudo apt-get install libxml2-dev libxslt-dev
sudo pip install --upgrade distribute
```

### 安装PHP sandbox
```
cd /opt
sudo git clone git://github.com/mushorg/BFR.git
cd BFR
sudo phpize
sudo ./configure --enable-bfr
sudo make && sudo make install
```
在`/etc/php/5/cli/php.ini`添加
`zend_extension = /usr/lib/php5/20090626+lfs/bfr.so`
或者
`zend_extension = /usr/lib64/php/modules/bfr.so`

### 安装Glastopf
`sudo pip install glastopf`
或者编译安装
```
sudo git clone https://github.com/mushorg/glastopf.git
cd glastopf
sudo python setup.py install
```

### 配置运行
`sudo glastopf-runner`
在目录下会产生db, data, log文件夹和一个glastopf.cfg配置文件，可以配置ip和端口，注意端口不要冲突
db存放本地sqlite数据库文件
运行`glastopf-runner`
浏览器访问Web
可以看到如下输出
```
2013-03-14 08:34:08,129 (glastopf.glastopf) Initializing Glastopf using "/opt/myhoneypot" as work directory.
2013-03-14 08:34:08,130 (glastopf.glastopf) Connecting to main database with: sqlite:///db/glastopf.db
2013-03-14 08:34:08,152 (glastopf.modules.reporting.auxiliary.log_hpfeeds) Connecting to feed broker.
2013-03-14 08:34:08,227 (glastopf.modules.reporting.auxiliary.log_hpfeeds) Connected to hpfeed broker.
2013-03-14 08:34:11,265 (glastopf.glastopf) Glastopf started and privileges dropped.
2013-03-14 08:34:32,853 (glastopf.glastopf) 192.168.10.85 requested GET / on 192.168.10.102
2013-03-14 08:34:32,960 (glastopf.glastopf) 192.168.10.85 requested GET /style.css on 192.168.10.102
2013-03-14 08:34:33,021 (glastopf.glastopf) 192.168.10.85 requested GET /favicon.ico on 192.168.10.102
```

### 参考
> https://github.com/mushorg/glastopf/blob/master/docs/source/installation/installation_ubuntu.rst
> http://seccentral.blogspot.com/2013/02/how-to-install-glastopf-on-centos-6-in.html

<script>pangu.spacingPage();</script>