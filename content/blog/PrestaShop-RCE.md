---
title: PrestaShop后台远程代码执行漏洞分析(CVE-2018-19126)
date: 2018-12-19 16:48:02
tags: [php,unserialize,prestashop,phpggc,phar]
categories: Security
---

<script src="https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/pangu.js"></script>

#### 0x01 概述

> PrestaShop是一款针对web2.0设计的全功能、跨平台的免费开源电子商务解决方案，自08年1.0版本发布，短短两年时间，发展迅速，全球已超过四万家网店采用Prestashop进行部署。Prestashop基于Smarty引擎编程设计，模块化设计，扩展性强，能轻易实现多种语言，多种货币浏览交易，支持Paypal等几乎所有的支付手段，是外贸网站建站的佳选。Prestashop是目前为止，操作最简单，最人性化，用户体验最佳的电子商务解决方案之一。 

11月7日 PrestaShop 官方发布新版本，修复两个高危漏洞，其中[CVE-2018-19126](https://nvd.nist.gov/vuln/detail/CVE-2018-19126)允许攻击者通过上传精心构造的文件导致任意代码执行，[CVE-2018-19125](https://nvd.nist.gov/vuln/detail/CVE-2018-19125)则导致攻击者删除服务器上的默认图片上传文件夹。

公告：http://build.prestashop.com/news/prestashop-1-7-4-4-1-6-1-23-maintenance-releases/

补丁：https://github.com/PrestaShop/PrestaShop/pull/11287

由于该漏洞触发需要后台上传权限，在这个系统中有上传权限的角色包括物流员、翻译者、销售人员，所以漏洞影响面还是有限。

#### 0x02 影响版本

> 1.6.x before 1.6.1.23
>
> 1.7.x before 1.7.4.4 

#### 0x03 环境搭建

下载`1.7.4.3`版本并安装

https://assets.prestashop2.com/en/system/files/ps_releases/prestashop_1.7.4.3.zip

在安装过程中会强制要求修改默认后台路径`admin`，同时要求删除`install`文件夹，从安全角度来讲，这是一个好设计：）

此案例中后台路径命名为`/admin-rename/`

#### 0x04 漏洞分析

根据[补丁](https://github.com/PrestaShop/PrestaShop/pull/11287/commits/4c6958f40cf7faa58207a203f3a5523cc8015148)位置显示漏洞出现在后台处理文件上传的模块，在`admin-dev/filemanager/ajax_calls.php`的`image_size`case分支，新版本直接删除了这段代码，其中最值得怀疑的是`getimagesize()`函数，在10月份披露的国外轻量级开源论坛系统`Vanilla Forums`就是因为该函数导致了一个远程代码执行漏洞，详细可以看[这里](https://srcincite.io/blog/2018/10/02/old-school-pwning-with-new-school-tricks-vanilla-forums-remote-code-execution.html)

那么`getimagesize()`函数存在什么问题呢？在今年的`BlackHat`大会上由`Sam Thomas`分享的反序列化漏洞议题主要讲了被忽略的`phar://`协议导致的`phar`反序列化漏洞，此前也简单介绍了一下这个议题，参考[由PHPGGC理解PHP反序列化漏洞](https://kylingit.com/blog/%E7%94%B1phpggc%E7%90%86%E8%A7%A3php%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96%E6%BC%8F%E6%B4%9E/)，在这里面提到一些在解析`phar://`协议时会产生风险的常用函数，如下图

![1539063733837](https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/images/1539063733837.png)

`getimagesize()`同样存在这个问题，当调用`getimagesize('phar://some/phar.ext');`的时候，`php`解析`phar`文件时会进行反序列化，如果其内容是恶意构造的，就能达到任意代码执行的效果 

风险点找到了，接下来看一下如何触发漏洞

`ajax_calls.php`在`filemanager`模块下面，通过`http://host/admin-rename/filemanager/dialog.php`页面调用，这个页面主要功能就是上传文件，以及创建文件夹，文件排序、删除等等，通过`action`参数控制操作，所以可以直接通过`action=image_size`访问到漏洞触发点

在`dialog.php`开头设置了一个`verify`字段

```php
$_SESSION["verify"] = "RESPONSIVEfilemanager";
```

而在页面检查了这个字段的值，所以无法直接访问`ajax_calls.php`页面，必须先访问`dialog.php`，目的应该是为了保证文件操作都是从`dialog.php`页面进行的吧

```php
if ($_SESSION['verify'] != 'RESPONSIVEfilemanager') {
    die('Forbidden');
}
```

然后我们就可以上传一个`phar`文件，当然系统限制文件后缀只能在白名单内`'jpg', 'jpeg', 'png', 'gif', 'bmp', 'tiff', 'svg', 'pdf', 'mov', 'mpeg', 'mp4', 'avi', 'mpg', 'wma', 'flv', 'webm' `，所以需要将`phar`文件重命名符合要求的后缀。由于在`phar://`解析时只要满足`phar`文件标识，即文件头必须以`__HALT_COMPILER();?>`结尾，所以并不限制文件后缀。此处也有一个技巧，我们可以创建一个合法的`jpeg`文件，同时又是一个`phar`文件，这个文件甚至可以绕过`MIME`检查，关于这个技巧可以参考[这里](https://www.nc-lp.com/blog/disguise-phar-packages-as-images)

接下来具体看`image_size`分支

```php
case 'image_size':
    if (realpath(dirname(_PS_ROOT_DIR_.$_POST['path'])) != realpath(_PS_ROOT_DIR_.$upload_dir)) {
        die();
    }
    $pos = strpos($_POST['path'], $upload_dir);
    if ($pos !== false) {
        $info = getimagesize(substr_replace($_POST['path'], $current_path, $pos, strlen($upload_dir)));
        echo json_encode($info);
    }

    break;
```

第一个`if`条件，检查`path`参数的绝对路径是否和系统定义的`upload_dir`绝对路径相等，`upload_dir`的值是

```php
$upload_dir = Context::getContext()->shop->getBaseURI().'img/cms/'; // path from base_url to base of upload folder (with start and final /)
```

而我们要`post`的`path`参数是这个样子`phar://path/phar.jpg`，显然无法通过判断。这时候考虑一下，假如我们把默认上传路径修改了，比如改成`img/test/`，那么系统就会找不到`img/cms/`这个路径，`realpath`返回结果为`false`，那么就可以绕过这个条件。

那么什么办法可以修改这个路径呢？

在`admin-rename/filemanager/execute.php`文件可以看到有一些文件及文件夹操作，允许用户删除或者重命名文件夹，通过`action`和`name`参数我们可以将默认的`img/cms/`修改成自定义文件夹，甚至可以删除这个路径，这也就是`CVE-2018-19125`这个漏洞的触发位置。

接下来看第2个条件

```php
$pos = strpos($_POST['path'], $upload_dir);
```

此处只要让`path`参数包含`img/cms/`字符串即可，这样经过后面的替换和拼接，`path`就类似于`phar://img/test/phar.pdf/var/www/html/img/cms/ `，不影响`phar`解析

`phar`文件最终进入`getimagesize()`，如果序列化一个可以执行任意代码的类，生成恶意的`phar`文件，构造一条完整的`POP`链，就可以形成一个`RCE`

在`PrestaShop`项目中存在`Monolog`，这是`php`下一个日志记录类库， 在这个库中的`BufferHandler`类的`handle`函数有一段存在风险的代码

```php
public function handle(array $record){
    //...
	if ($this->processors) {
    	foreach ($this->processors as $processor) {
        	$record = call_user_func($processor, $record);
    	}
	}
    //...
}
```

如果`$this->processors`和`$record`均可控的话，就可以造成一个命令执行，所以我们可以序列化这个类构造`POP`链

#### 0x05 漏洞利用

##### 一、生成phar文件

我们利用之前介绍过的[PHARGCC](https://github.com/s-n-t/phpggc)工具生成一个包含`POP`链的`phar`文件，选择`Monolog/RCE1`，看一下这个`gadget`的使用

```shell
> pharggc -i Monolog/RCE1
Name           : Monolog/RCE1
Version        : 1.18 <= 1.23
Type           :
Vector         : __destruct

./pharggc Monolog/RCE1 <code>
```

生成一个`out.phar`，并重命名成`out.png`

```shell
> pharggc Monolog/RCE1 "phpinfo();"
Payload written to: out.phar
```

然后通过`http://host/admin-rename/filemanager/dialog.php`上传到服务器上

##### 二、重命名默认上传目录

通过`http://host/admin-rename/filemanager/execute.php?action=rename_folder`重命名默认上传目录

![](https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/images/20181220111941.png)

##### 三、发送payload

通过`path`参数传入上传的`phar`文件，`getimagesize()`自动解析`phar`，触发反序列化

![](https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/images/20181220103714.png)

#### 0x06 补丁分析

补丁位置：https://github.com/PrestaShop/PrestaShop/pull/11287

![](https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/images/20181220105842.png)

官方对这两处漏洞做了修复，一方面是直接删除了`case 'image_size'`分支，一方面也严格检查了文件`mime`类型，同时对`realpath`检查时限制了必须是已存在的目录

![](https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/images/20181220110721.png)

![](https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/images/20181220110815.png)

#### 0x07 总结

这个漏洞触发流程并不复杂，几个限制也能简单地绕过，关键在于某些函数没有考虑到传入`phar://`的情况，在解析该协议时产生反序列化漏洞，近段时间以来基于`phar`的漏洞也在逐渐增加，希望能够引起开发者的重视。



<script>pangu.spacingPage();</script>

