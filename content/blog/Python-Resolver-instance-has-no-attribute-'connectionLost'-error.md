---
title: Python 解析 DNS 时 Resolver instance has no attribute 'connectionLost' 异常解决
date: 2016-08-26 16:32:18
tags: Python
categories: Coding
---
<script src="https://ob5vt1k7f.qnssl.com/pangu.js"></script>

某个项目中用到dns相关的模块，在长时间运行后偶尔抛出异常:
![resolver_error](https://ob5vt1k7f.qnssl.com/dns_error.png)

`Resolver instance has no attribute 'connectionLost'`
```
Unhandled Error
Traceback (most recent call last):
  File "dns.py", line 174, in <module>
    reactor.run()
  File "/usr/lib/python2.7/dist-packages/twisted/internet/base.py", line 1169, in run
    self.mainLoop()
  File "/usr/lib/python2.7/dist-packages/twisted/internet/base.py", line 1181, in mainLoop
    self.doIteration(t)
  File "/usr/lib/python2.7/dist-packages/twisted/internet/pollreactor.py", line 167, in doPoll
    log.callWithLogger(selectable, _drdw, selectable, fd, event)
--- <exception caught here> ---
  File "/usr/lib/python2.7/dist-packages/twisted/python/log.py", line 84, in callWithLogger
    return callWithContext({"system": lp}, func, *args, **kw)
  File "/usr/lib/python2.7/dist-packages/twisted/python/log.py", line 69, in callWithContext
    return context.call({ILogContext: newCtx}, func, *args, **kw)
  File "/usr/lib/python2.7/dist-packages/twisted/python/context.py", line 118, in callWithContext
    return self.currentContext().callWithContext(ctx, func, *args, **kw)
  File "/usr/lib/python2.7/dist-packages/twisted/python/context.py", line 81, in callWithContext
    return func(*args,**kw)
  File "/usr/lib/python2.7/dist-packages/twisted/internet/posixbase.py", line 599, in _doReadOrWrite
    self._disconnectSelectable(selectable, why, inRead)
  File "/usr/lib/python2.7/dist-packages/twisted/internet/posixbase.py", line 260, in _disconnectSelectable
    selectable.readConnectionLost(f)
  File "/usr/lib/python2.7/dist-packages/twisted/internet/tcp.py", line 257, in readConnectionLost
    self.connectionLost(reason)
  File "/usr/lib/python2.7/dist-packages/twisted/internet/tcp.py", line 433, in connectionLost
    Connection.connectionLost(self, reason)
  File "/usr/lib/python2.7/dist-packages/twisted/internet/tcp.py", line 277, in connectionLost
    protocol.connectionLost(reason)
  File "/usr/lib/python2.7/dist-packages/twisted/names/dns.py", line 1908, in connectionLost
    self.controller.connectionLost(self)
exceptions.AttributeError: Resolver instance has no attribute 'connectionLost'
```
<!-- more -->

查阅相关资料发现不是自己代码的问题，而是 `Twisted` 库中的 `twisted.names.client.Resolver` 类没有 `connectionLost` 方法，而这个方法本身并不需要做任何事，于是解决办法就是，找到 `twisted.names.client.Resolver`，在最后添加 `connectionLost` 方法：
```
def connectionLost(self, p):
    pass
```
异常解决。

另外，还遇到
```
Traceback (most recent call last):
Failure: twisted.names.error.DNSQueryTimeoutError:
```
异常，这个也很奇怪，因为一开始并没有出现，而是运行了一段时间后对某些特定的查询会出现，解决办法是导入dns查询超时异常类，然后捕捉该异常
`from twisted.names.error import DNSQueryTimeoutError`


参考:

> https://twistedmatrix.com/trac/ticket/5224
> http://stackoverflow.com/questions/15944617/handle-error-on-a-simple-dns-twisted-client

<script>pangu.spacingPage();</script>


