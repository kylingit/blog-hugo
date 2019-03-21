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

![](https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/images/20190221233754.png)

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

### 0x04 LFI to RCE

到目前为止我们可以把含有恶意代码的图片写入任意目录，下一步就是想办法包含这个文件。

在`Wordpress`中，访问一篇文章或者任意页面，都需要从数据库取出相应的模板文件位置并由浏览器渲染出来。注意到上面截图，`wp_postmeta`数据库中有个字段名称为`_wp_page_template`，这个字段用来保存加载页面所需要的模板文件，默认为`default`，`wordpress`程序根据需要加载的页面类型从当前主题下选择需要的模板，例如访问一篇单独的文章，这个过程会拼凑出文件名并检查主题下的这些文件是否存在，如果存在则包含进来，相关方法是`locate_template()`和`load_template()`

![](https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/images/20190225145624.png)

![](https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/images/20190225145650.png)

搜索发现实现从数据库取出`_wp_page_template`变量的方法是`get_page_template_slug()`

![](https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/images/20190225150445.png)

接着发现调用`get_page_template_slug()`方法的`get_single_template()`方法，其最后返回的是查找模板函数，即`get_query_template()`

![](https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/images/20190225150545.png)

而正是在`get_query_template()`中，执行了定位模板文件的操作

![](https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/images/20190225150712.png)

至此一条利用链就串起来了，利用第一个漏洞覆盖数据库中的`_wp_page_template`值，修改为包含恶意代码的图片所在路径，在页面加载的过程中`wordpress`查询并定位该文件，包含后造成代码执行。

`Wordpress`中处理图片相关的库有两个，分别是`Imagick`和`GD`，优先选择使用`Imagick`，而`Imagick`处理图片时不处理`EXIF`信息，因此可以把恶意代码设置在`EXIF`部分，经过裁剪后会保留`EXIF`信息，此时再进行包含就能造成代码执行。

在选择相应图片库处理图片时，如果此时加载的是`Imagick`，在`$editor->load()`时会创建`Imagick()`对象，然后尝试读取远程图片地址。此时需要注意的是，高版本的`Imagick`库不支持远程链接，测试`Imagick-6.9.7`版本正常创建并写入图片

```php
$implementation = _wp_image_editor_choose( $args );

if ( $implementation ) {
    $editor = new $implementation( $path );
    $loaded = $editor->load();

    if ( is_wp_error( $loaded ) )
        return $loaded;

    return $editor;
}
```

```php
$this->image = new Imagick();
//...
$this->image->readImage( $filename );
```

复现：

1.上传图片，更新描述信息并保存，抓包修改`meta_input[_wp_attached_file]`，目录穿越至当前主题文件夹

![](https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/images/20190226172313.png)

2.裁剪图片并在主题文件夹下生成裁剪后图片

![](https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/images/20190226173532.png)

3.上传一个附件，更新描述信息并抓包，修改`meta_input[_wp_page_template]`，加载模板的时候自动包含该图片，代码执行成功

![](https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/images/20190226171510.png)



### 0x05 关于mkdir

在漏洞调试过程中最后一步`$editor->save( $dst_file )`过程，最终执行到的是`wp_mkdir_p()`方法中的`mkdir`函数

```php
mkdir( $target, $dir_perms, true)
```

关于`mkdir()`函数，需要注意的是`mode`参数和`recursive`参数，分别代表了创建的文件夹权限和是否递归创建，这两个参数的不同导致在`Linux`平台和`Windows`平台的结果不一致

![](https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/images/20190222093526.png)

在上面漏洞链中，进入最终`mkdir()`的参数是这样的

```php
mkdir( 'D:\phpStudy\PHPTutorial\WWW\wordpress-4.9.8/wp-content/uploads/2019/02/admin.jpeg?../../../1', 511, true)
```

![](https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/images/20190222094608.png)

单独把`path`拿出来测试，在第三个参数`recursive`分别为`true`和`false`时，测试结果如下

![](https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/images/20190222095108.png)

这里导致结果不一致是因为Windows下文件夹对`?`的处理，当指定递归创建模式时，系统会尝试创建名为`admin.jpeg?..`的目录，又因为Windows下的目录不能含有`?`，因此`recursive=true`时是创建失败的，导致`wordpress`最终生成图片也无法成功。而在Linux下可以没有`?`的限制，`payload`可以成功触发。

要想在`Windows`下利用漏洞，一个技巧是利用`#`字符，`#`在`url`中表示为网页位置指定标识符，只在浏览器中起作用，对解析资源时是忽略后面的字符的，因此在`wordpress`中两个方式尝试获取图片资源时同样会出现不一致，导致漏洞产生。

### 0x06 PoC

见上面分析

### 0x07 总结

在分析过程中踩了不少坑，每一个都浪费了不少时间，简单记录避免再次踩中。主要的有这么几个：

1. `Wordpress`自动更新；
2. 需要手动修改触发裁剪函数的`action`；
3. `mkdir`创建文件夹时特殊字符的问题；
4. `Imagick`读取远程文件的问题；

这个漏洞主要成因在于我们可以通过参数传递任意值覆盖数据库中的字段，从而引入`../`构成目录穿越，在裁剪图片后保存文件时并没有对文件目录做检查，造成目录穿越漏洞，最终可以写入恶意图片被包含或者通过`Imagick`漏洞触发远程代码执行，利用链挺巧妙，值得学习。



参考：

- https://blog.ripstech.com/2019/wordpress-image-remote-code-execution/
- https://github.com/WordPress/WordPress/commit/43bdb0e193955145a5ab1137890bb798bce5f0d2#diff-c3d5c535db5622f3b0242411ee5f9dfd

<script>pangu.spacingPage();</script>