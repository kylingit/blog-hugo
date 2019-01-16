---
title: ThinkPHP 5.0.x-5.0.23、5.1.x、5.2.x 全版本远程代码执行漏洞分析
date: 2019-01-12 14:18:20
tags: [vul,sec,thinkphp,rce]
categories: Security
---

<script src="https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/pangu.js"></script>

### 0x01 概述

1月11日，`ThinkPHP`官方发布新版本`5.0.24`，在1月14日和15日又接连发布两个更新，这三次更新都修复了一个安全问题，该问题可能导致远程代码执行 ，这是`ThinkPHP`近期的第二个高危漏洞，两个漏洞均是无需登录即可远程触发，危害极大。

- 公告

  https://blog.thinkphp.cn/910675

  http://blog.nsfocus.net/thinkphp-5-0-5-0-23-rce/

### 0x02 影响版本

> ThinkPHP 5.0.x ~ 5.0.23
>
> ThinkPHP 5.1.x ~ 5.1.31
>
> ThinkPHP 5.2.0beta1

### 0x03 环境搭建

选择`5.0.22`完整版和`5.1.31`版本进行复现分析

### 0x04 漏洞分析

#### 一、`5.0.x`版本

我们知道可以通过`http://127.0.0.1/public/index.php?s=index`的方式通过`s`参数传递具体的路由，具体调用如下

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

可以看到在进入`self::exec($dispatch, $config)`前，`$dispatch`的值是通过

`$dispatch = self::routeCheck($request, $config)`

设置的，这时候如果`debug`模式开启，就会调用`$request->param()`，也就是下面`exec()`中会调用到的函数，经过下面分析就能发现，在`debug`模式开启时就能直接触发漏洞，原理是一样的。

进入`exec()`方法看一下：

![](https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/images/20190112153147.png)

`exec()`方法根据`$dispatch`的值选择进入不同的分支，当进入`method`分支时，调用`Request::instance()->param()`方法，跟进`param()`，看到调用了`Request`类的`method()`方法 ：

![](https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/images/20190114120946.png)

其中`method()`方法就是补丁修改的位置，在这个方法中，如果`method`等于`true`，则调用`$this->server()`方法：

![](https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/images/20190112153449.png)

在`server()`方法中调用`$this->input`方法：

![](https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/images/20190112154611.png)

接着调用了`filterValue()`方法：

![](https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/images/20190112154923.png)

而`filterValue()`则调用了`call_user_func()`函数，如果两个参数均可控，则会造成命令执行：

![](https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/images/20190112175411.png)

此时的调用栈如下：

![](https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/images/20190114104035.png)

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

`$dispatch` 的值通过`routeCheck()`方法设置，跟进`routeCheck()`方法：

![](https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/images/20190112155407.png)

调用了`check()`方法：

![](https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/images/20190112155704.png)

`check()`方法中根据不同的`$rules`值返回不同的结果，而`$rules`的值由`$method`决定，`$method`则由`$request->method()`返回值取小写获得，所以再次回到`$request->method()`方法，这次没有参数

![](https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/images/20190112153449.png)

如果`$method`不等于`true`，则会取配置选项`var_method`，该值为`_method`

![](https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/images/20190112153759.png)

然后调用`$this->{$this->method}($_POST);`语句，此时假设我们控制了`$method`的值，也就意味着可以调用`Request`类的任意方法，而当调用构造方法`__construct()`时，就可以覆盖`Request`类的任意成员变量，也就是上面分析的`$this->filter`和`$this->server`两个值，同时也可以覆盖`$this->method`，直接指定了`check()`方法中的`$method`值。

##### 1. 构造`PoC`

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

##### 2. 流程图

整个漏洞的调用流程图如下所示：

![](https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/images/20190116110438.png)



#### 二、`5.1.x`/`5.2.x`版本

在`5.1`和`5.2`版本上，这个变量覆盖依然存在，我们同样可以通过`_method`参数覆盖`var_method`，并最终执行到`Request::input()`方法，通过`array_walk_recursive`把传入的数组传给回调函数`filterValue`，最终也是在`filterValue`中完成命令执行，具体调用如下

当传入`_method`参数为`filter`时，覆盖了`Request`原始的`filter`成员，在经过路由检查进入`Request::instance()->param()`方法时，经过`$this->method(true)调用，`返回的`$method`值为`POST`，于是进入`post`分支，调用`input()`方法，由于第一个参数为空，返回我们传入的`post`值

![](https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/images/20190116095414.png)

然后把数组合并到`$this->param`，接着再次调用`input()`方法，经过`$this->getFilter`返回`filter`值，由于此时`$data`是一个数组(即`$this->param`)，于是进入`if`分支，经过`array_walk_recursive()`函数把数组传给回调函数`filterValue`，遍历键值后同样由`call_user_func`完成命令执行

```php
if (is_array($data)) {
    array_walk_recursive($data, [$this, 'filterValue'], $filter);
    if (version_compare(PHP_VERSION, '7.1.0', '<')) {
        // 恢复PHP版本低于 7.1 时 array_walk_recursive 中消耗的内部指针
        $this->arrayReset($data);
    }
} else {
    $this->filterValue($data, $name, $filter);
}
```

##### 1. 构造`PoC`

```php
a=system&b=id&_method=filter
```

需要在程序加入忽略异常提示：

```php
error_reporting(0);
```

调用栈如图

![](https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/images/20190116095551.png)

##### 2. 流程图

`5.1.x`版本的漏洞调用流程图如下所示：

![](https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/images/20190116104019.png)

### 0x05 补丁分析

![](https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/images/20190114134401.png)

在三个版本的更新补丁中，限制了`$this->method`为`GET`，`POST`，`DELETE`，`PUT`，`PATCH`这几个方法，因此不能从外部传入方法名再调用`Request`类的任意方法或是覆盖原有变量。

补丁链接：

- 5.0.24：

  https://github.com/top-think/framework/commit/4a4b5e64fa4c46f851b4004005bff5f3196de003

- 5.1.31：

  https://github.com/top-think/framework/commit/2454cebcdb6c12b352ac0acd4a4e6b25b31982e6

- 5.2-beta.2：

  https://github.com/top-think/framework/commit/7c24500e463704583e0778b7ec6efce607ddef5f

### 0x06 总结

这三漏洞本质上都是变量覆盖漏洞，在一处存在缺陷的方法中没有对用户输入做严格判断，通过传递`_method`参数覆盖了配置文件的`_method`，导致可以访问`Request`类的任意函数，而在`Request`的构造函数中又创建了恶意的成员变量，导致后面的命令执行；而在`5.1`和`5.2`版本中则是直接覆盖了过滤器，在忽略运行异常的情况下会触发漏洞，整个利用链可以说是非常巧妙了。



<script>pangu.spacingPage();</script>