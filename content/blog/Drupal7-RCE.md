---
title: CVE-2018-7600 Drupal 7.x 版本代码执行漏洞分析
date: 2018-04-20 23:05:34
tags: [vul,sec,Drupal]
categories: Security
---
#### 0x01 概述
CVE-2018-7600影响范围包括了Drupal 6.x，7.x，8.x版本，前几天8.x版本的PoC出来之后大家都赶紧分析了一波，然后热度似乎慢慢退去了。两天前[Drupalgeddon2](https://github.com/dreadlocked/Drupalgeddon2)项目更新了7.x版本的exp，实际环境也出现了利用，下面就简单来看一下

看到项目上这样写
> Drupal < 7.58 ~ user/password URL, attacking triggering_element_name form & #post_render parameter, using PHP's passthru function

提示了问题出在`user/password`路径下，通过`#post_render`传递恶意参数，问题出现在`triggering_element_name`表单处理下

#### 0x02 漏洞分析
我们从三个问题入手，为什么PoC发了两个包，第二次请求为什么要带上一个`form_build_id`，以及为什么选择`user/password`这个入口

先分析第一个post，照例还是先看一下Drupal 7的表单处理流程，跟8版本不太一样，但是入口还是相似的。
根据文档描述，当我们提交一个表单(例如找回密码)时，系统会通过`form_builder()`方法创建一个form
![user/passwd](https://ob5vt1k7f.qnssl.com/lffx4.jpg)
一系列预处理后，会由`drupal_build_form()`方法创建一个表单，在第386行调用`drupal_process_form()`方法，
跟进`drupal_process_form()`方法，这时候默认的`$form_state['submitted']`为false
![submitted](https://ob5vt1k7f.qnssl.com/dd7fx.png)
不满足if条件，`$form_state['submitted']`被设置为true
![true](https://ob5vt1k7f.qnssl.com/kqijr.png)
于是进入这个分支，最终被`drupal_redirect_form`重定向
![drupal_redirect_form](https://ob5vt1k7f.qnssl.com/so5l4.jpg)

我们的目的是要让系统缓存一个`form_build_id`，要想form被缓存，就得想办法让`if ($form_state['submitted'] && !form_get_errors() && !$form_state['rebuild'])`不成立，也就是说要使`$form_state['submitted']`为false
从而进入下面的`drupal_rebuild_form`

那么如何让`$form_state['submitted']`为false呢？
在`includes/form.inc`第886行
`$form = form_builder($form_id, $form, $form_state);`
跟进`form_builder`方法，第1987行

```php
if (!empty($form_state['triggering_element']['#executes_submit_callback'])) {  $form_state['submitted'] = TRUE;}
```
当`$form_state['triggering_element']['#executes_submit_callback']`存在值的时候就为true，那么我们就想办法让这个值为空
往上看第1972行

```php
if (!$form_state['programmed'] && !isset($form_state['triggering_element']) && !empty($form_state['buttons'])) {  $form_state['triggering_element'] = $form_state['buttons'][0];}
```
如果没有设置`$form_state['triggering_element']`，那么`$form_state['triggering_element']`就设置为第一个button的值，所以正常传递表单的时候`$form_state['triggering_element']['#executes_submit_callback']`就总会有值
![button](https://ob5vt1k7f.qnssl.com/1do73.jpg)

现在问题来了，如何构造一个form能够确保`$form_state['triggering_element']['#executes_submit_callback']`为空或者说不存在这个数组呢？

我们注意到第1864行

```php
if (!empty($element['#input'])) {  _form_builder_handle_input_element($form_id, $element, $form_state);}
```
`_form_builder_handle_input_element()`方法对表单先进行了处理，跟进去看一下

第2144行

```php
// Determine which element (if any) triggered the submission of the form and// keep track of all the clickable buttons in the form for// form_state_values_clean(). Enforce the same input processing restrictions// as above.if ($process_input) {  // Detect if the element triggered the submission via Ajax.  if (_form_element_triggered_scripted_submission($element, $form_state)) {    $form_state['triggering_element'] = $element;  }
```
这里`$form_state['triggering_element']`被设置为`$element`，前提是满足`_form_element_triggered_scripted_submission()`方法，继续跟入
第2180行

```php
function _form_element_triggered_scripted_submission($element, &$form_state) {  if (!empty($form_state['input']['_triggering_element_name']) && $element['#name'] == $form_state['input']['_triggering_element_name']) {    if (empty($form_state['input']['_triggering_element_value']) || $form_state['input']['_triggering_element_value'] == $element['#value']) {      return TRUE;    }  }  return FALSE;}}```
这个方法的意思是说如果`_triggering_element_value`和`$element`的键值都相等的话，返回true
`$form_state['triggering_element']`赋值为`$element`，其中不含`['#executes_submit_callback']`，一开始的条件就成立了

根据PoC，我们传入`_triggering_element_name=name`

![element](https://ob5vt1k7f.qnssl.com/3c6kh.jpg)
看到进入这个分支，进入`form_set_cache()`方法
![](https://ob5vt1k7f.qnssl.com/cn1jh.jpg)
![](https://ob5vt1k7f.qnssl.com/lskgu.png)
数据库中插入缓存`form_build_id`
![](https://ob5vt1k7f.qnssl.com/odfo8.png)
成功写入缓存


接下去来看一下这个缓存有什么用
分析PoC的第二个包，请求参数是这样`q=file/ajax/name/%23value/form_build_id`
`form_build_id`即我们上一个写入数据库的缓存表单

首先请求会进入`includes/menu.inc`的`menu_get_item()`方法，

```php
function menu_get_item($path = NULL, $router_item = NULL) {  $router_items = &drupal_static(__FUNCTION__);  if (!isset($path)) {    $path = $_GET['q'];  }  if (isset($router_item)) {    $router_items[$path] = $router_item;  }  if (!isset($router_items[$path])) {    // Rebuild if we know it's needed, or if the menu masks are missing which    // occurs rarely, likely due to a race condition of multiple rebuilds.    if (variable_get('menu_rebuild_needed', FALSE) || !variable_get('menu_masks', array())) {      if (_menu_check_rebuild()) {        menu_rebuild();      }    }    $original_map = arg(NULL, $path);    $parts = array_slice($original_map, 0, MENU_MAX_PARTS);    $ancestors = menu_get_ancestors($parts);    $router_item = db_query_range('SELECT * FROM {menu_router} WHERE path IN (:ancestors) ORDER BY fit DESC', 0, 1, array(':ancestors' => $ancestors))->fetchAssoc();    if ($router_item) {      // Allow modules to alter the router item before it is translated and      // checked for access.      drupal_alter('menu_get_item', $router_item, $path, $original_map);      $map = _menu_translate($router_item, $original_map);      $router_item['original_map'] = $original_map;      if ($map === FALSE) {        $router_items[$path] = FALSE;        return FALSE;      }      if ($router_item['access']) {        $router_item['map'] = $map;        $router_item['page_arguments'] = array_merge(menu_unserialize($router_item['page_arguments'], $map), array_slice($map, $router_item['number_parts']));        $router_item['theme_arguments'] = array_merge(menu_unserialize($router_item['theme_arguments'], $map), array_slice($map, $router_item['number_parts']));      }    }    $router_items[$path] = $router_item;  }  return $router_items[$path];}```
`$path`即我们传进去的q参数，经过一系列处理传给`menu_get_ancestors()`方法，该方法把path重新组合成一堆router，也就是Drupal处理路由到具体url的传参方式，最终被`db_query_range()`带入数据库查询
我们关注查询结果`$router_item`的`page_callback`值，因为这个值最终会作为参数被带入`call_user_func_array()`

```php
if ($page_callback_result == MENU_SITE_ONLINE) {  if ($router_item = menu_get_item($path)) {    if ($router_item['access']) {      if ($router_item['include_file']) {        require_once DRUPAL_ROOT . '/' . $router_item['include_file'];      }      $page_callback_result = call_user_func_array($router_item['page_callback'], $router_item['page_arguments']);    }    else {      $page_callback_result = MENU_ACCESS_DENIED;    }  }  else {    $page_callback_result = MENU_NOT_FOUND;  }}```
![call_user_func_array](https://ob5vt1k7f.qnssl.com/qvqwz.png)
到这里就跟8版本的情况有点类似了
跟入回调函数`file_ajax_upload()`
![file_ajax_upload](https://ob5vt1k7f.qnssl.com/tij37.jpg)
还是一样，把`$form_parents`完整取出赋值给`$form`，加上一些前缀后缀后最终进入`drupal_render()`方法

最终得到执行
![passthru](https://ob5vt1k7f.qnssl.com/rpwrs.jpg)

到目前为止我们分析清楚了为什么PoC要发两次包，以及第二次请求为什么要带上一个`form_build_id`，现在来想一想为什么要请求`user/password`这个路径呢？
在user这个module下的`user_pass()`方法

```php
function user_pass() {  global $user;  $form['name'] = array(    '#type' => 'textfield',    '#title' => t('Username or e-mail address'),    '#size' => 60,    '#maxlength' => max(USERNAME_MAX_LENGTH, EMAIL_MAX_LENGTH),    '#required' => TRUE,    '#default_value' => isset($_GET['name']) ? $_GET['name'] : '',  );
  ...
  return $form;```
看到这里是不是感觉跟8版本很相似，`#default_value`从get的`name`参数里取值，而name可以作为数组传入，它的属性在下面正好可以被利用，一个巧妙的利用链就串起来了。

#### 总结
Drupal 7.x的利用比8.x要复杂一些，但触发点和一开始的风险因素还是类似的，一是接收参数过滤不当，而是可控参数进入危险方法。官方补丁把入口处的`#`全给过滤了，简单粗暴又有效，估计再利用框架本身的特性想传递进一些数组或元素就很难了。

