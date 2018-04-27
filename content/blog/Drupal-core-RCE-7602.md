---
title: CVE-2018-7602 Drupal 内核远程代码执行漏洞分析
date: 2018-04-26 17:21:11
tags: [vul,sec,Drupal]
categories: Security
---

#### 0x01 概述
4月25日，Drupal官方发布通告，Drupal Core 存在一个远程代码执行漏洞，影响 7.x 和 8.x 版本。据分析，这个漏洞是`CVE-2018-7600`的绕过利用，两个漏洞原理是一样的，通告还称，已经发现了这个漏洞和`CVE-2018-7600`的在野利用，详情请看 https://www.drupal.org/sa-core-2018-004

#### 0x02 影响版本
Drupal 6.x，7.x，8.x

修复版本
Drupal 7.59，Drupal 8.4.8，Drupal 8.5.3

#### 0x03 环境搭建
历史版本
https://www.drupal.org/project/drupal/releases

#### 0x04 漏洞分析
分析还是以7.57版本为例。跟7600漏洞的7.x版本很相似，只不过入口不一样，可以参考http://blog.nsfocus.net/cve-2018-7600-drupal-7-x/

上回漏洞的关键点是让系统缓存一个`form_build_id`，这个form存着我们传入的恶意参数，第二个请求从中取出来然后执行。
这次的原理还是一样，触发漏洞还是需要发两个post包，一个存入`form_build_id`一个取出后执行。

这次的问题出在删除文章的时候，因此需要文章删除权限，我们先走一遍正常删除文章的逻辑

![delete](https://ob5vt1k7f.qnssl.com/VGL3Y)
请求中每个node即代表一篇文章。
可以看到是会重定向到文章页面的，根据上个漏洞的分析我们猜测，一定还是走到了`drupal_redirect_form()`，我们已经知道如果走到`drupal_redirect_form()`分支，是不会往数据库缓存`form_build_id`的，我们的目的还是让程序不满足一定条件从而不进行表单提交后重定向，所以还是跟着`CVE-2018-7600`的套路来走

从代码层面看一下

之前的流程还是一样，直接跳到`drupal_build_form()`方法第386行
```
drupal_process_form($form_id, $form, $form_state);
```
跟入`drupal_process_form()`

![drupal_process_form](https://ob5vt1k7f.qnssl.com/w0cQZ)
还是一样，`$form_state['submitted']`被设置为true

回到902行

`if ($form_state['submitted'] && !form_get_errors() && !$form_state['rebuild'])`
条件被满足，进入这个分支便会执行`drupal_redirect_form()`

![drupal_redirect_form](https://ob5vt1k7f.qnssl.com/aoYne)

而在这一步之前需要经过的判断是`_form_element_triggered_scripted_submission()`
所以回到一开始的问题，构造一个`_triggering_element_value`使得键值对相等，从而不进行rebuild

我们传入`_triggering_element_name=form_id`

![post](https://ob5vt1k7f.qnssl.com/UBCWM)
![form_id](https://ob5vt1k7f.qnssl.com/Rbrm4)
可以看到条件被满足，`$form_state['submitted']`没有被设置为true，还是保持默认值false

![submitted](https://ob5vt1k7f.qnssl.com/UYLtu)
![submitted](https://ob5vt1k7f.qnssl.com/o98bE)

进入`drupal_rebuild_form()`

![drupal_rebuild_form](https://ob5vt1k7f.qnssl.com/WdeFV)
表单被缓存

![form_set_cache](https://ob5vt1k7f.qnssl.com/MCCPJ)
![cache_form](https://ob5vt1k7f.qnssl.com/U5Tej)

然后我们发送第二个post包来取出我们构造好的form，向**`file/ajax/actions/cancel/%23options/path`**发起请求

![post2](https://ob5vt1k7f.qnssl.com/voiue)

参数传递进去

![file_ajax_upload](https://ob5vt1k7f.qnssl.com/luJMk)
最终还是跟入到
`$output = drupal_render($form);`
根据前几次的经验，我们还是选择`'#post_render'`参数，

![post_render](https://ob5vt1k7f.qnssl.com/GV3Rc)
假如我们能控制这个参数，在`drupal_render()`方法里就会把这个参数作为`$function`函数名，而传给它的参数则是`[%23markup]`

所以问题回到了一开始，我们需要传递什么样的恶意参数，可以让系统直接接收而不经过过滤，还是之前的套路，搜索module下删除文章的相关操作

![node_form_delete_submit](https://ob5vt1k7f.qnssl.com/niMWz)
可以看到`node_form_delete_submit()`方法从get方法直接接收参数`destination`，与最初分析正常删除文章的参数正是同一个，那么我们就可以利用`destination`传进恶意参数

构造如下
**`destination=a?q[%2523post_render][]=passthru%26q[%23type]=markup%26q[%23markup]=dir`**

`a`参数是次要的，主要是`q`参数，因为在`includes/common.inc`的`drupal_parse_url()`方法

```php
if (isset($options['query']['q'])) {
    $options['path'] = $options['query']['q'];
    unset($options['query']['q']);
  }
```
从q取出值赋给`$options['path']`，也就是a被覆盖了，这个时候的`$options['path']`就是我们传入的数组

注意q的元素需要转义百分号，对`#`进行二次编码，以绕过`CVE-2018-7600`的补丁，不然在取值时会被认为`q[`是一个值

![options](https://ob5vt1k7f.qnssl.com/s4o7s)
参数缓存进整个form后通过第二个请求取出，同样经过

```php
foreach ($form_parents as $parent) {
    $form = $form[$parent];
  }
```
遍历叶子节点取出参数

![parent](https://ob5vt1k7f.qnssl.com/Wgz3w)
进入`drupal_render()`执行

![passthru](https://ob5vt1k7f.qnssl.com/SsmNM)

#### 0x05 PoC
略

#### 0x06 补丁
7.x的补丁

https://github.com/drupal/drupal/commit/080daa38f265ea28444c540832509a48861587d0

![patch](https://ob5vt1k7f.qnssl.com/2Zhjq)
其中一个重要操作就是对`destination`参数进行了净化

![cleanDestination](http://ob5vt1k7f.qnssl.com/CaZPF)

#### 0x07 总结
总的来说这个漏洞是CVE-2018-7600的另一个利用点，只是入口方式不一样，最终执行点还是相同的，所以还是那句话，一旦参数可控并且没有经过正确的过滤，就很有可能出问题。













