---
title: ThinkPHP 5.0.x 版本远程代码执行漏洞分析
date: 2019-01-12 14:18:20
tags: [vul,sec,thinkphp,rce]
categories: Security
---

<script src="https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/pangu.js"></script>

### 0x01 概述

1月11日，`ThinkPHP`官方发布新版本5.0.24，修复了一个安全问题，该问题可能导致`GetShell`，这是`ThinkPHP`近期的第二个高危漏洞，两个漏洞均是无需登录即可远程触发，危害极大。

公告：https://blog.thinkphp.cn/910675

补丁：https://github.com/top-think/framework/commit/4a4b5e64fa4c46f851b4004005bff5f3196de003

### 0x02 影响版本

> 5.0.x ~ 5.0.23

### 0x03 环境搭建

选择`5.0.22`版本进行复现分析

### 0x04 漏洞分析

我们知道可以通过`http://127.0.0.1/public/index.php?s=captcha`的方式通过`s`参数传递具体的路由，具体调用如下

`index.php`

```php
require __DIR__ . '/../thinkphp/start.php';
```

`start.php`

```php
App::run()->send();
```

跟进`run()`方法

![](https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/images/20190112152929.png)

可以看到在进入`self::exec($dispatch, $config)`前，`$dispatch`的值是通过`$dispatch = self::routeCheck($request, $config)`设置的，先进入`exec()`方法看一下：

![](https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/images/20190112153147.png)

`exec()`方法根据`$dispatch`的值选择进入不同的分支，当进入`method`分支时，调用`Request::instance()->param()`方法，跟进`param()`，看到调用了`Request`类的`method()`方法 ：

```php
$method = $this->method(true);
```

其中`method()`方法就是补丁修改的位置，在这个方法中，如果`method`等于`true`，则调用`$this->server()`方法：：

![](https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/images/20190112153449.png)

在`server()`方法中调用`$this->input`方法：

![](https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/images/20190112154611.png)

接着调用了`filterValue()`方法：

![](https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/images/20190112154923.png)

而`filterValue()`则调用了`call_user_func()`方法，如果两个参数均可控，则会造成命令执行：

![](https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/images/20190112175411.png)

回头看一下`$filter`和`$value`参数从哪里来：

- `$filter`：

```php
$filter = $this->getFilter($filter, $default);
```

在`getFilter()`中设置了`$filter`值：

```php
$filter = $filter ?: $this->filter;
```

也即由`$this->filter`决定

- `$value`

`$value`为第一个参数`$data`，即为传入数组的值，由`$this->server`决定

所以最终的问题就是如何从请求中传入`$this->filter`和`$this->server`这两个值，构造`call_user_func()`的参数触发漏洞。



回到最开始的`run()`方法，其中：

```php
$dispatch = self::routeCheck($request, $config);
```

`$dispatch` 的值通过`routeCheck()`方法设置，根据routeCheck()方法：

![](https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/images/20190112155407.png)

调用了`check()`方法：

![](https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/images/20190112155704.png)

`check()`方法中根据不同的`$rules`值返回不同的结果，而`$rules`的值由`$method`决定，`$method`则由`$request->method()`返回值取小写获得，所以再次回到`$request->method()`方法，这次没有参数

![](https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/images/20190112153449.png)

如果`$method`不等于`true`，则会取配置选项`var_method`，该值为`_method`

![](https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/images/20190112153759.png)

然后调用`$this->{$this->method}($_POST);`语句，此时假设我们控制了`$method`的值，也就意味着可以调用`Request`类的任意方法，而当调用构造方法`__construct()`时，就可以覆盖`Request`类的任意成员变量，也就是上面分析的`$this->filter`和`$this->server`两个值，同时也可以覆盖`$this->method`，直接指定了`check()`方法中的`$method`值。

### 0x05 构造PoC

首先要主动触发`Request`类的构造函数，通过参数`_method=__construct`传入，进入到`__construct`方法，该方法把参数遍历并设置值：

![](https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/images/20190112163221.png)

所以我们可以传入**`filter=system`**来设置`$this->filter`的值

> 此处`filter`不是数组也可以，因为在`getFilter()`中虽然对`filter`是字符串的情况进行了按`,`分割，但是传入一个值的情况下不影响最终的返回值

再看`$this->server`，在调用`$this->server('REQUEST_METHOD')`时指定了键值，所以通过传入`server`数组即可

**`server[REQUEST_METHOD]=id`**

然后我们注意到上面`check()`方法，

`$rules = isset(self::$rules[$method]) ? self::$rules[$method] : [];`

它的返回值由`$rules`决定，而`$rules`的值取决于键值`$method`，当我们指定`$method`为`get`时，可以正确获取到路由信息，从而通过`checkRoute()`检查，此时我们通过指定**`method=get`**覆盖`$this->method`的值即可

![](https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/images/20190112173807.png)



最终的`PoC`：

`_method=__construct&filter=system&method=get&server[REQUEST_METHOD]=id`

![](https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/images/20190112163746.png)

调用栈如下图所示

![](https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/images/20190112175904.png)



### 0x06 总结

这个漏洞本质上是一个覆盖漏洞，通过`_method`覆盖了配置文件的`_method`，导致可以访问`Request`类的任意函数，而在`Request`的构造函数中又创建了恶意的成员变量，导致后面的命令执行，整个漏洞利用可以说是非常巧妙了。



<script>pangu.spacingPage();</script>





