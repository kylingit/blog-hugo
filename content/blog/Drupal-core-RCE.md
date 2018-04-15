---
title: CVE-2018-7600 Drupal 内核远程代码执行漏洞分析
date: 2018-04-13 23:05:34
tags: [vul,sec,Drupal]
categories: Security
---

#### 0x01 概述
https://www.drupal.org/sa-core-2018-002

#### 0x02 影响版本
Drupal 6.x，7.x，8.x

修复版本
Drupal 7.58，Drupal 8.5.1

#### 0x03 环境搭建
历史版本
https://www.drupal.org/project/drupal/releases


#### 0x04 流程梳理
先来理清一下Drupal处理表单的情况。更详细的可以看[文档](http://www.thinkindrupal.com/node/1100)

> Drupal提供了一个应用程序接口（API），用来生成、验证和处理HTML表单。表单API将表单抽象为一个嵌套数组，里面包含了属性和值。在生成页面时，表单呈现引擎会在适当的时候将数组呈现出来。

> 模块使用关联数组向Drupal描述表单。Drupal的表单引擎负责为要显示的表单生成HTML，并使用三个阶段来安全的处理提交了的表单：验证、提交、重定向。

Drupal比较特殊，它不像大部分cms通过html直接渲染页面，而是把接收的数据交给`core/lib/Drupal/Core/Form/FormBuilder.php`的`buildForm()`方法处理，`buildForm()`经过处理后返回一个结构体(数组)，数组通过引擎生成HTML。

当我们提交一个表单(例如注册页面)，`buildForm()`方法会根据`$form_id`取出数据，经过一系列处理后返回一个树形结构，这个结构就是通过数组存储的，就是我们看到的类似`[$current_file_count]['#attributes']['class'][]`的结构，数组每个元素作为一个叶子节点，后续就把整个`form`结构渲染出页面。

当我们在注册页面上传一张图片的时候，`form`结构被传给`core/modules/file/src/Element/ManagedFile.php`的`uploadAjaxCallback()`方法，这个方法用来处理上传文件的情况
```php
 public static function uploadAjaxCallback(&$form, FormStateInterface &$form_state, Request $request) {
    /** @var \Drupal\Core\Render\RendererInterface $renderer */
    $renderer = \Drupal::service('renderer');

    $form_parents = explode('/', $request->query->get('element_parents'));

    // Retrieve the element to be rendered.
    $form = NestedArray::getValue($form, $form_parents);

    // Add the special AJAX class if a new file was added.
    $current_file_count = $form_state->get('file_upload_delta_initial');
    if (isset($form['#file_upload_delta']) && $current_file_count < $form['#file_upload_delta']) {
      $form[$current_file_count]['#attributes']['class'][] = 'ajax-new-content';
    }
    // Otherwise just add the new content class on a placeholder.
    else {
      $form['#suffix'] .= '<span class="ajax-new-content"></span>';
    }

    $status_messages = ['#type' => 'status_messages'];
    $form['#prefix'] .= $renderer->renderRoot($status_messages);
    $output = $renderer->renderRoot($form);

    $response = new AjaxResponse();
    $response->setAttachments($form['#attached']);

    return $response->addCommand(new ReplaceCommand(NULL, $output));
  }
```

![upload](https://ob5vt1k7f.qnssl.com/8b1t3)

![form_parents](https://ob5vt1k7f.qnssl.com/Q3ys0)

问题就出现在`$request->query->get('element_parents')`这个地方，`$form_parents`父节点的值是从`get()`取出`element_parents`参数传进去的，进入下面的`NestedArray::getValue()`方法，`getValue()`的作用是接收一个节点，把这个节点下的叶子节点全部遍历出来，再根据叶子节点的`key-value`值进行后续操作。

![getValue](https://ob5vt1k7f.qnssl.com/39ciX)
![user_picture](https://ob5vt1k7f.qnssl.com/f8qiO)

按理说这样的功能很正常，关键就在于这个`element_parents`正是我们可以控制的，也就是说我们可以指定`uploadAjaxCallback()`渲染我们给它的参数，而这个参数可以是恶意的。

#### 0x05 漏洞分析
那么我们传进去什么参数呢？我们先来测试一下，正常注册流程，`mail`参数传进去一个数组的话会怎么样

![mail](https://ob5vt1k7f.qnssl.com/KJE50)
可以看到我们构造的“子节点”被存储在`mail-value`下，如果要取出这个值就得让上面提到的`getValue()`接收这个参数，所以我们构造`element_parents=account/name/%23value`，这样子`getValue()`就会遍历出我们构造的参数

现在参数已经能够传进去了，那么在哪里执行呢？继续往下跟
```pytho
$current_file_count = $form_state->get('file_upload_delta_initial');
if (isset($form['#file_upload_delta']) && $current_file_count < $form['#file_upload_delta']) {
	$form[$current_file_count]['#attributes']['class'][] = 'ajax-new-content';
}
// Otherwise just add the new content class on a placeholder.
else {
	$form['#suffix'] .= '<span class="ajax-new-content"></span>';
}

$status_messages = ['#type' => 'status_messages'];
$form['#prefix'] .= $renderer->renderRoot($status_messages);
$output = $renderer->renderRoot($form);
```
可以看到经过`getValue()`遍历出来的叶子节点(就是此时的`form`)被传进`$renderer->renderRoot()`方法，跟进去看一下

`core/lib/Drupal/Core/Render/Renderer.php`
```
  public function render(&$elements, $is_root_call = FALSE) {
...
    try {
      return $this->doRender($elements, $is_root_call);
    }
    catch (\Exception $e) {
      // Mark the ::rootRender() call finished due to this exception & re-throw.
      $this->isRenderingRoot = FALSE;
      throw $e;
    }
  }
```
调用`doRender()`方法

![doRender](http://ob5vt1k7f.qnssl.com/6Htxw)
这个方法比较长，但是我们从中找到了几处执行`call_user_func()`的地方，先看一下第三处
```php
if (isset($elements['#post_render'])) {
    foreach ($elements['#post_render'] as $callable) {
        if (is_string($callable) && strpos($callable, '::') === FALSE) {
            $callable = $this->controllerResolver->getControllerFromDefinition($callable);
        }
        $elements['#children'] = call_user_func($callable, $elements['#children'], $elements);
    }
}
```
接收的第一个参数`$elements['#post_render']`作为函数，第二个参数`$elements['#children']`作为参数，在上面被赋值
```php
if (!$theme_is_implemented && isset($elements['#markup'])) {
    $elements['#children'] = Markup::create($elements['#markup'] . $elements['#children']);
}
```
这两个参数都是我们可控的，于是造成一个代码执行

![call_user_func](https://ob5vt1k7f.qnssl.com/jHlV8)


回头看一下这处`call_user_func_array`，这里的`$callable`和`$args`两个参数实际上也是可控的，通过`#lazy_builder`属性传进来，checkpoint的分析报告正是分析了这个地方

![call_user_func_array](http://ob5vt1k7f.qnssl.com/QlR23)

#### 0x06 总结
关注这个漏洞也是好长时间了，当时粗略看了一下，因为补丁直接对入口进行了过滤，要找到真正触发的地方太难了，所以也迟迟不见PoC出来。checkpoint的分析报告出来后好好跟了一遍，不得不感叹人家真厉害(逃...

这个漏洞关键点有两个，一个是`uploadAjaxCallback`里`$form_parents`由get直接传进参数，这里就存在风险；
另一处`call_user_func`两个参数均可控，两者结合造成一个严重的远程代码执行漏洞，看分析报告如何一步步构造利用链，可谓是十分精彩了。



#### 0x07 参考
- https://research.checkpoint.com/uncovering-drupalgeddon-2/
- https://github.com/a2u/CVE-2018-7600













