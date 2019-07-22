---
title: SA-CORE-2019-008 Drupal访问绕过漏洞分析
date: 2019-07-19 10:27:05
tags: [vul,sec,Drupal]
categories: Security
---

<script src="https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/pangu.js"></script>

### 0x01 概述

7月17日，Drupal官方发布Drupal核心安全更新公告，修复了一个访问绕过漏洞，攻击者可以在未授权的情况下发布/修改/删除文章，CVE编号`CVE-2019-6342`

公告地址：<https://www.drupal.org/sa-core-2019-008>

### 0x02 受影响的版本

-   Drupal Version == 8.7.4

### 0x03 漏洞复现

安装`Drupal 8.7.4`版本，登录管理员账户，进入后台`/admin/modules`，勾选`Workspaces`模块并安装

![](https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/images/20190719101447.png)

在页面上方出现如下页面则安装成功，管理员可以切换`Stage`模式或者`Live`模式

![](https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/images/20190719104359.png)

另外开启一个浏览器访问首页（未登录任何账户），访问<http://127.0.0.1/drupal-8.7.4/node/add/article>

可直接添加文章，无需作者或管理员权限。

![](https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/images/20190719104407.png)

受影响操作包括基本文章操作（添加、修改、删除、上传附件等）

### 0x04 漏洞分析

#### Workspaces的功能

`Workspaces`是`Drupal 8.6`核心新增的实验模块，主要功能是方便管理员一次性发布/修改多个内容。

`Workspaces`有两种模式，分别为`Stage`模式和`Live`模式，，默认为`Live`模式，两者的区别在于：

-   `Stage`模式下修改内容不会及时更新，所有文章修改完毕后管理员可以通过`Deploy to Live`发布到实际环境，相当于一个暂存区；

![](https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/images/20190719104412.png)

-   `Live`下更新是即时的，发布后站点内容立即更新。

在这两种模式下，由于编码失误导致存在一个缺陷：匿名用户无需登录即可创建/发布/修改/删除文章，问题点出现在权限鉴定模块`EntityAccess`下。

#### 漏洞分析

当用户发起请求时，会根据当前操作回调相关权限检查模块对当前用户权限进行检查，请求调用为事件监听器(`EventListener`)的`RouterListener`类，在其`onKernelRequest()`方法中调用`AccessAwareRouter`类的`matchRequest()`方法，随后调用`AccessManager->checkRequest()`方法，最后在`AccessManager->performCheck()`方法中通过`call_user_func_array`回调对应的操作进入到具体的操作权限检查

![](https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/images/20190719104431.png)

例如发布文章时回调的是`access_check.node.add`，相关方法在`NodeAccessControlHandler`控制器中定义，这个控制器继承自`EntityAccessControlHandler`，在父类的`createAccess()`方法中回调对应操作的`create_access`权限，过程中会拼接上模块名和相应钩子作为回调函数，

```php
$function = module . '_' . $hook
```

例如此处回调的是`workspaces_entity_create_access()`方法，进入到Workspaces中。

![](https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/images/20190719104438.png)

在调用`entityCreateAccess()`方法时有一个关键操作`bypassAccessResult`

![](https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/images/20190719104445.png)

`bypassAccessResult()`方法是一个检查用户是否有`"绕过节点访问权限(bypass node access)"`的操作，是Workspaces中特有的，这个方法决定了"如果用户在各自的激活的工作区中，那么他将拥有所有权限"，这里的所有权限指文章相关的增删改操作。

这个权限虽然奇怪但确实是一个设计好的功能，正常操作应该在后台`admin/people/permissions`中配置好用户是否拥有这个权限，默认情况下匿名用户和认证用户都没有权限

![](https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/images/20190719104451.png)

当开启了`Bypass content entity access in own workspace`权限后用户才可以在未登录的情况下发布/删除文章，而此次漏洞就绕过了这个配置，默认情况下进行了越权操作。

具体分析一下`bypassAccessResult()`的实现，整个过程返回的是`AccessResultAllowed`对象或者`AccessResultNeutral`对象，所谓"中立"是因为后续还可能会对结果再做判断，但在这个漏洞中其实就是`access`和`forbidden`的区别：

![](https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/images/20190719104457.png)

首先获取了当前激活的工作区，然后通过`allowedIf`判断当前用户是否有权限，随后这些数据存入缓存，包括缓存内容、缓存标签和过期时间。然后再经过一次`allowedIfHasPermission`判断，这个方法的作用是，如果权限不对就设置一个`reason`，在这个漏洞中没有起到作用，到目前为止权限校验都是正常的，在没有配置后台工作区匿名权限的时候，返回的是一个`AccessResultNeutral`对象，也就是"禁止"。

接下来就是出现问题的地方

```php
$owner_has_access->orIf(access_bypass);
```

通过补丁可以发现漏洞就修补了这行语句，把`orIf`换成了`andIf`

![](https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/images/20190719104509.png)

这两个方法的设计逻辑比较复杂，最主要的功能是对一个如果返回为"中立"的结果做后续判断，如果采用orIf方法合并，那么是否允许由调用者决定；如果以andIf方法合并，则被当做禁止。

具体到此次漏洞上的区别如下方图片所示：

-   `orIf()`

![](https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/images/20190719104515.png)

返回的是`AccessResultAllowed`对象

![](https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/images/20190719104520.png)

-   `andIf()`

![](https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/images/20190719104525.png)

返回的是`AccessResultNeutral`对象

![](https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/images/20190719104529.png)

在检查完毕后会回到`AccessAwareRouter->checkAccess()`方法，在该方法中对返回结果进行了判断，`AccessResultNeutral`的`isAllowed()`返回`false`，因此会抛出异常 

![](https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/images/20190719104536.png)

返回到页面上则是`Access denied`

![](https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/images/20190719104542.png)

更新补丁后只有在开启后台匿名用户权限后才能进行文章操作，该选项默认不开启。

相关调用栈为

```
Drupal\workspaces\EntityAccess->bypassAccessResult()

Drupal\workspaces\EntityAccess->entityCreateAccess()

...

Drupal\Core\Extension\ModuleHandler->invokeAll()

Drupal\node\NodeAccessControlHandler->createAccess()

Drupal\node\Access\NodeAddAccessCheck->access()

Drupal\Core\Access\AccessManager->performCheck()

Drupal\Core\Routing\AccessAwareRouter->checkAccess()

Drupal\Core\Routing\AccessAwareRouter->matchRequest()

Symfony\Component\HttpKernel\EventListener\RouterListener->onKernelRequest()

...

DrupalKernel.php:693, Drupal\Core\DrupalKernel->handle()

index.php:19, {main}()
```



### 0x05 总结

此次漏洞出现在设计过程的一个疏忽，在默认没有分配权限的情况下用户可以绕过权限检查进行发布/删除/修改文章操作，但由于该漏洞仅影响Drupal 8.7.4版本，并且需要开启`Workspaces`模块，这又是一个实验功能，默认不启用，因此漏洞影响减弱了不少，用户可以升级`Drupal`版本或者关闭`Workspaces`模块以消除漏洞影响。



<script>pangu.spacingPage();</script>