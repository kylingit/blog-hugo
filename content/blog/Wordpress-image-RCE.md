---
title: Wordpress Image 远程代码执行漏洞分析
date: 2019-02-21 14:45:49
tags: [vul,sec,wordpress,rce]
categories: Security
---

<script src="https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/pangu.js"></script>

### 0x01 概述

2月20日，RIPS披露了`Wordpress`内核`Image`模块相关的一个高危漏洞，该漏洞由目录穿越和文件包含组成，最终可导致远程代码执行，目前还没有PoC披露。

从`RIPS`描述的细节来看，漏洞出现在`wordpress`编辑图片时，由于没有过滤`Post Meta` 值导致可以修改数据库中`wp_postmeta`表的任意字段，而在加载本地服务器上的文件时没有对路径进行过滤，导致可以传递目录穿越参数，最终保存图片时可以保存至任意目录。当某个主题include了某目录下的文件时，便可以造成代码执行。

### 0x02 环境搭建

该漏洞影响`4.9.9`版本以下的`wordpress`程序，`4.9.9`引入了过滤函数，对用户输入的`post data`进行了检查，不合法的参数被过滤，主要修改如下图：

![](https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/images/20190221162810.png)

值得注意的是，在安装低版本时，安装过程中会自动更新核心文件，因此旧版本的`wp-admin/includes/post.php`会更新至最新版本，所以安装过程中可以删除自动更新相关模块，或者离线安装。

### 0x03 漏洞分析

#### 漏洞一：数据覆盖

漏洞出现在wordpress媒体库裁剪图片的过程，当我们上传图片到媒体库时，图片会被保存至`wp-content/uploads/yyyy/mm`目录，同时会在数据库中wp_postmeta表插入两个值，分别是`_wp_attached_file`和`_wp_attachment_metadata`，保存了图片位置和属性相关的序列化信息。

![](https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/images/20190221164008.png)

当我们修改图片属性（例如修改标题或者说明）的时候，`admin-media-Edit more details` 会调用`wp-admin/includes/post.php`的`edit_post()`方法，该方法的参数全部来自于`$_POST`，没有进行过滤

![](https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/images/20190221163553.png)

然后会调用到`update_post_meta()`方法，该方法根据`$post_ID`修改`post meta field`，接着调用`update_metadata()`更新`meta`数据，完成之后更新`post`数据，调用`wp_update_post()`方法

![](https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/images/20190221165642.png)

 在`wp_update_post()`方法中，如果`post_type=attachment`，则进入`wp_insert_attachment()`，接着调用`wp_insert_post()`，在`wp_insert_post()`方法中判断了`meta_input`参数，如果传入了该参数，就遍历数组用来更新`post_meta`

![](https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/images/20190221172343.png)

进入`update_post_meta()`，调用`update_metadata()`，在`update_metadata()`方法中对数据库进行更新操作，而在整个过程中对键值没有任何过滤，意味着我们可以传入指定的key来设置它的值，调用栈如下图所示

![](https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/images/20190221165806.png)



于是构造数据包更新数据库中`_wp_attached_file`的值，插入一个包含`../`的值，以便在下面触发目录遍历。

![](https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/images/20190221183004.png)



![](https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/images/20190221174154.png)

这是第一个漏洞——通过参数覆盖了数据库数据，在补丁处正是对`meta_input`这个参数做了过滤，如果包含则通过对比`array`舍弃该参数。

#### 漏洞二：目录遍历

接着寻找一个获取`_wp_attached_file`的值并进行了文件操作相关的方法。

在`wordpress`的`图片裁剪`功能中，有这样的功能：

1. 图片存在于`wp-content\uploads\yyyy\mm`目录，则从该目录读取图片，修改尺寸后另存为一张图片；
2. 如果图片在该目录不存在，则通过**本地**服务器下载该图片，如从`http://127.0.0.1/wordpress/wp-content/uploads/2019/02/admin.jpeg`下载，裁剪后重新保存。

这个功能是为了方便一些插件动态加载图片时使用。

然而因为本地读取和通过`url`读取的差异性，导致可以构造一个带参数的`url`，如`http://127.0.0.1/wordpress/wp-content/uploads/2019/02/admin.jpeg?1.png`，在本地读取时会发现找不到`admin.jpeg?1.png`，而远程获取时会忽略`?`后面的参数部分，照样获取到`admin.jpeg`，裁剪后保存。如果构造的url包含路径穿越，例如`http://127.0.0.1/wordpress/wp-content/uploads/2019/02/admin.jpeg?../../1/1.png`，`wordpress`将裁减后的图片保存至指定的文件夹，当图片包含恶意代码被引用时，就可能造成代码执行。

图片裁剪功能在`wp_crop_image()`方法中，但是该方法不能在页面中触发，需要手动更改相应的`action`

首先在页面裁剪图片，并点击保存

![1550743458074](E:\Document\Blog\content\blog\assets\1550743458074.png)

抓取数据包：

```
action=image-editor&_ajax_nonce=4c354c778b&postid=5&history=%5B%7B%22c%22%3A%7B%22x%22%3A0%2C%22y%22%3A5%2C%22w%22%3A347%2C%22h%22%3A335%7D%7D%5D&target=all&context=edit-attachment&do=save
```

`post body`包含了相应的`action`和`context`，以及供还原文件的历史文件大小，此处需要修改`action`为`crop-image`以便触发`wp_crop_image()`方法，相关调用如下

在`wp-admin/admin-ajax.php`定义了裁剪图片的操作

![](https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/images/20190221180906.png)

判断了用户权限和`action`名称后调用`do_action`，最终在`apply_filters()`中进入`wp_crop_image()`:

![](https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/images/20190221181053.png)

![](https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/images/20190221181232.png)

进入`wp_ajax_crop_image()`方法，在这个方法中进行了多项判断，全部符合才能进入裁剪图片方法，如下图注释所示

![](https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/images/20190221181644.png)

首先计算`nonce`和`expected`值并对比，如果不一致就验证不通过，相关方法是`check_ajax_referer()`-->`wp_verify_nonce()`。注意到传入`check_ajax_referer()`的`$attachment_id`参数，该参数取自`$_POST['id']`，并参与后面的`expected`计算，因此当我们直接更改`action=crop-image`是无法通过校验的，需要传入`id`的，即为`postid`的值。

在进入`wp_crop_image()`时还需要传递裁剪后的图片宽度和高度信息，所以还需要增加c`ropDetails[dst_width]`和`cropDetails[dst_height]`两个参数。


`wp_crop_image()`方法如下


![](https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/images/20190221183701.png)

从数据库取出`_wp_attached_file`后并没有做检查，形如`2019/02/admin.jpeg?../../1.png`的文件无法被找到，于是进入`_load_image_to_edit_path()`通过`wp_get_attachment_url()`方法生成本地`url`

![](https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/images/20190221184123.png)

随后实例化一个`WP_Image_Editor`用来裁剪并生成裁剪后的图片，之后调用`wp_mkdir_p()`方法创建文件夹，含有`../`的参数进入该方法后同样没有经过过滤，最终执行到`mkdir`创建文件夹

```php
mkdir( $target, $dir_perms, true)
```

此时的`target`值是这个样子，穿越目录后在`2019`目录下创建`1`文件夹，并生成`cropped-1.png`文件

```
D:\phpStudy\PHPTutorial\WWW\wordpress-4.9.8/wp-content/uploads/2019/02/admin.jpeg?../../../1
```



注意：此处有一个坑，我们观察上面的`url`，在`mkdir`的时候会把`admin.jpeg?../`作为一个目录，而在Windows下的目录不能出现`?`，所以上面的payload在Windows下无法成功，经过测试，`#`可以存在于Windows目录，因此在Windows下的payload如下所示：

```
meta_input[_wp_attached_file]=2019/02/admin.jpeg#../../../1/1.png
```

写入数据库中即为`2019/02/admin.jpeg#../../../1/1.png`

最终构造第二个数据包触发裁剪图片并保存：

![](https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/images/20190221182857.png)

最终在指定目录下生成裁剪后的图片文件，以`cropped-`作为前缀

![](https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/images/20190221232341.png)



这样子我们可以制作一张图片马，在主题文件夹下生成，或者指定任意目录，被`include`后即可造成代码执行。

### 0x04 PoC

见上面分析

### 0x05 总结

这个漏洞主要成因在于我们可以通过参数传递任意值覆盖数据库中的字段，从而引入`../`构成目录穿越，在裁剪图片后保存文件时并没有对文件目录做检查，造成目录穿越漏洞，最终可以写入恶意图片被包含或者通过`Imagick`漏洞触发远程代码执行，利用链挺巧妙，值得学习。



参考：

- https://blog.ripstech.com/2019/wordpress-image-remote-code-execution/
- https://github.com/WordPress/WordPress/commit/43bdb0e193955145a5ab1137890bb798bce5f0d2#diff-c3d5c535db5622f3b0242411ee5f9dfd

<script>pangu.spacingPage();</script>