---

title: SA-CORE-2019-004 Drupal内核从XSS到RCE漏洞分析
date: 2019-04-22 15:19:04
tags: [vul,sec,Drupal]
categories: Security
---

<script src="https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/pangu.js"></script>

### 0x01 概述

3月20日，Drupal官方发布`SA-CORE-2019-004`漏洞预警，修复了一处文件名处理异常，当我们上传特殊文件名时可以绕过限制在服务器上创建“无后缀”文件，精心构造的文件经过浏览器解析后可以触发XSS漏洞，再进一步可以达到代码执行的目的。

官方描述该漏洞为中危影响，因为该漏洞需要登录，并且需要作者权限来上传文件才能触发。

详细参考：<https://www.drupal.org/sa-core-2019-004>

### 0x02 漏洞分析

根据官方[补丁](https://github.com/drupal/core/commit/933f4f9d620af5807c4eb4ec17dc4eb4193a667c)，最主要修改了一处`preg_replace`异常，修改的方法为`file.inc:file_create_filename()`

![](https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/images/20190422165341.png)

该方法的主要功能是处理上传文件的路径，返回形如`public://2019-04/1.png`的路径，然后由`drupal`自身实现的方法将`public://`路径转换为相对路径。如果文件系统已经存在同名文件，则在文件名会追加`_0`, `_1`。问题出在这行处理`utf-8`编码的代码上

`$basename = preg_replace('/[\x00-\x1F]/u', '_', $basename);`

这行代码的意思的如果文件名中含有`0x00-0x1F`区间的字符时，将其转换为`_`。查看标准`ASCII`码表，字符所对应的十进制数范围为`0-127`，`0x00-0x1F`对应的数为`0-31`，也就是前32个不常见或者说不可显字符。**假如我们传入的字符范围大于128时，`preg_replace`便会出错**，返回一个`NULL`值，于是此时的`basename`便被赋值为空，而下面对`basename`也没有多余的操作，于是我们就得到了一个没有后缀的文件，而且这个文件的名称也是可以预测的，相关代码如下

```php
if (file_exists($destination)) {
    // Destination file already exists, generate an alternative.
    $pos = strrpos($basename, '.');
    if ($pos !== FALSE) {
      $name = substr($basename, 0, $pos);
      $ext = substr($basename, $pos);
    }
    else {
      $name = $basename;
      $ext = '';
    }

    $counter = 0;
    do {
      $destination = $directory . $separator . $name . '_' . $counter++ . $ext;
    } while (file_exists($destination));
  }
```

在开始有一个`file_exists`判断，所以我们需要上传相同文件至少两次，才能进入`if`分支。由于`$basename`为空，`$name`也被赋值为空，接下来进入一个循环，如果文件已经存在，就把文件命名为`路径+分隔符+name+_+counter+后缀`的形式，也就是同名文件会被加上后缀`_0` `_1`等，而此时表达式的值即为`下划线+计数器`的值，于是我们在`/sites/default/files/pictures/<YYYY-MM>/`目录下得到类似`_0 _1`的无后缀文件。

![](https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/images/20190422171620.png)

值得注意的是，上传的文件可以是任意类型，不局限于图片文件，可以通过`http://drupal-site/sites/default/files/<YYYY-MM>/_1`访问到文件。

### 0x03 尝试利用

#### 初探

此时我们想到，可以上传一个html文件，诱使管理员点击形成一个XSS漏洞。

但是这里存在一个问题，当通过`Home >> Add content >> 上传Image`上传文件时，如果传了一个纯HTML文件，是无法上传成功的，因为在`core/modules/file/file.module`里有一个检查

`$errors = file_validate($file, $validators);`

`file_validate()`方法通过下图4个方法检查图片有效性

![](https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/images/20190423111858.png)

其中`file_validate_is_image`方法使用了`ImageFactory`库，非图片文件无法通过这个检查，于是上传失败，也就无法在`sites/default/files/<YYYY-MM>/`生成文件。

当然这里是可以绕过的，那就是在上传的内容前加上图片文件头，例如`GIF`或`GIF89a`，能够绕过`file_validate_is_image`检查上传文件，然而这并没有什么用，因为访问`http://drupal-site/sites/default/files/<YYYY-MM>/_1`时，浏览器无法判断文件类型，所以不会解析...

也就是说

- 纯HTML文件无后缀，浏览器可以解析，但是无法上传；
- 增加图片文件头，可以上传，但是浏览器不能解析；

于是现在陷入一个两难境地，需要找个另外的点。

#### 突破

我们注意到在`新建文章`页面除了上传图片外，还有一个编辑器内置的图片上传接口，从这个接口分析一下

![](https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/images/20190423111342.png)

跟入相关方法，之前的流程都是一样的，同样会进入`file_validate()`方法，同样检测文件合法性，但是此时的`$validators`有所不一样：

![](https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/images/20190423111805.png)

`file_validate_is_image`变成了`file_validate_image_resolution`，这个方法是检测图片是否符合大小，但是对于非图片文件会直接忽略，返回一个空数组

![](https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/images/20190423113048.png)

方法说明里也说了会忽略非图片文件

> Non-image files will be ignored.

于是`file_validate()`不报错，校验通过，成功上传文件，目录是`sites/default/files/inline-images/`，访问文件成功触发

![](https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/images/20190423113606.png)

现在可以利用这个漏洞上传一个html文件，攻击面就扩大了许多，简单的可以是一个XSS，复杂的可以是个钓鱼页面(这个漏洞需要作者权限，故可以钓鱼管理员)，再进一步，如何进行命令执行甚至反弹一个shell呢？

#### 组合拳

联想到1月份Drupal官方修复的[SA-CORE-2019-002](https://www.drupal.org/sa-core-2019-002)漏洞，文件操作函数处理`phar`文件时会触发反序列化形成代码执行漏洞，在此处正好可以用上。`phar`反序列化风险影响几乎所有文件操作函数，而在Drupal中`File system`功能就存在这个缺陷，在设置本地临时文件夹的时候会进行路径检查，相关方法是`system_check_directory()`

![](https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/images/20190423135902.png)

在这个方法中存在`is_dir()`函数，当`is_dir()`函数处理`phar:// stream wrapper`时，便会触发反序列化，如果传入一个恶意构造的`phar`文件就可以造成代码执行。

![](https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/images/20190423140328.png)

![](https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/images/20190425174400.png)

因此现在的思路是，上传一个`phar`文件，诱使管理员点击链接把临时文件路径设置为`phar://test.phar`触发漏洞反弹shell

### 0x04 漏洞利用

通过上述分析，有几个限制需要突破：

1. 生成的phar文件不能通过图片上传接口上传，否则会失败；
2. 需要写一个html让管理员打开链接自动发送请求来修改临时文件路径；

因此漏洞利用分为以下几步：

1. 生成一个能反弹shell的phar文件；

    Drupal反序列化的POP链已经比较多了，可以参考这里[由 PHPGGC 理解 PHP 反序列化漏洞](https://kylingit.com/blog/%E7%94%B1phpggc%E7%90%86%E8%A7%A3php%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96%E6%BC%8F%E6%B4%9E/)

    选择`GuzzleHttp\Psr7`类，使用[PHARGGC](https://kylingit.com/blog/%E7%94%B1phpggc%E7%90%86%E8%A7%A3php%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96%E6%BC%8F%E6%B4%9E/)直接生成phar文件

    ![](https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/images/20190423143704.png)

2. 上传phar文件

    把`out.phar`修改为png后缀，通过编辑器的接口上传，获得文件路径

    `http://127.0.0.1/drupal-8.6.5/sites/default/files/inline-images/phar.png`

3. 生成一个html文件

    这个html文件需要让管理员打开链接时自动发送请求来修改临时文件路径，与`CSRF`非常相似，所以直接使用`burpsuite`抓包生成`CSRF PoC`

    ![](https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/images/20190423150512.png)

    

    然后增加一个自动点击提交表单的操作，省去手动submit

    ```html
    <script type="text/javascript">
        var a  = document.getElementById('form1')
        a.submit()
    </script>
    ```

    

4. 再通过编辑器的接口上传这个html文件，修改文件名的`ascii`值大于128。**注意需要上传至少两次**，以生成`_0`、`_1`文件

    ![](https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/images/20190423153014.png)

    

5. 访问`http://drupal-site/sites/default/files/inline-images/_0`时浏览器解析为html并自动提交表单，触发漏洞

![](https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/images/20190423153510.png)

这里因为反弹shell会把页面卡住，可以增加一个跳转首页，更隐蔽地触发漏洞。



### 0x05 总结

这个漏洞由一个`preg_replace()`引起，由于没有正确处理异常，导致可以上传“任意”文件；而`phar`反序列化漏洞在一年前就已经公布了，把几个漏洞组合在一起形成一条漂亮的攻击链，值得学习。站在管理员角度应该关注安全更新，及时更新应用，而对于开发者来说也要重视安全风险，不可忽视任何一处不起眼的安全隐患。



参考：

- <https://github.com/drupal/core/commit/933f4f9d620af5807c4eb4ec17dc4eb4193a667c>

- <https://www.zerodayinitiative.com/blog/2019/4/11/a-series-of-unfortunate-images-drupal-1-click-to-rce-exploit-chain-detailed>
- <https://paper.seebug.org/897/>





<script>pangu.spacingPage();</script>

