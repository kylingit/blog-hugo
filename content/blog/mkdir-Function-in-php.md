---
title: 对PHP中的mkdir()函数的研究
date: 2019-03-20 15:27:20
tags: [php, kernel, debug]
categories: Research
notoc: false
---

<script src="https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/pangu.js"></script>

### 0x01 缘起

在前阵子分析[WORDPRESS IMAGE 远程代码执行漏洞](https://kylingit.com/blog/wordpress-image-%E8%BF%9C%E7%A8%8B%E4%BB%A3%E7%A0%81%E6%89%A7%E8%A1%8C%E6%BC%8F%E6%B4%9E%E5%88%86%E6%9E%90/)的过程中，在文末提到一点关于`php`中的`mkdir()`函数，在触发漏洞时这个地方存在一点疑惑，即当`mkdir()`第三个参数分别为`false`和`true`时，分别是能成功创建文件夹和创建失败，后来有同学发现和他的测试结果有偏差，两种情况都无法创建，在互相确认了`php`版本后，对`mkdir()`函数进行了深入的研究，发现里面大有文章。

当时的测试结果是这样的，环境是`Windows`+`php-7.0.12-nts`，在`recursive=false`时成功穿越目录并创建了文件夹

![](https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/images/20190222095108.png)



本文重新编译了php方便调试，版本是`php-7.2.16-ts`和`php-7.2.16-nts`，测试结果如下

![](https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/images/20190320161208.png)

![](https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/images/20190320161651.png)

可以看到只有在非线程安全下并且`recursive=false`时才成功创建，总结如下表所示

|       Windows       |   **thread-safe**   | **non-thread safe** |
| :-----------------: | :-----------------: | :-----------------: |
| **recursive=false** |   fail (No error)   |     **success**     |
| **recursive=true**  | fail (Invalid path) | fail (Invalid path) |

接下来从源码角度看看`php`如何实现`mkdir()`函数，探究一下为何会出现差异

### 0x02 调试

用`Visual Studio 2017`打开项目，定位到`php-7.2.16-src/main/streams/plain_wrapper.c`line 1234，方法`php_plain_files_mkdir()`即`mkdir()`的实现，在此处下个断点，然后运行脚本，接着选择`调试-附加到进程`，选择编译好的`php.exe`进程，成功命中断点。

### 0x03 源码分析

#### 1. recursive=true

##### thread-safe

首先分析在`recursive=true`的情况，跟随断点来看一下`php_plain_files_mkdir()`这个方法

![](https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/images/20190320162930.png)

看到对`recursive`进行了判断，进了不同的分支，分别执行`php_mkdir()`和`expand_filepath_with_mode()`。`recursive=true`时进入`expand_filepath_with_mode()`

![](https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/images/20190320163530.png)

这个`expand_filepath_with_mode()`方法会判断当前路径是相对路径还是绝对路径，然后把路径传入`virtual_file_ex()`，如果是相对路径的话会在该方法中拼接成完整的路径，随后进行一个重要的判断

![](https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/images/20190320165438.png)

如果是`Windows`系统且路径中包含了`*`或`?`，则直接返回错误，这也就是为什么在复现`wordpress`漏洞时构造的`PoC`中含有`?`无法创建目录的原因(`wordpress`指定了`recursive=true`)，当时使用`#`绕过了这个限制

回到上面，`virtual_file_ex()`没有通过验证，最终抛出的异常是`"Invalid path"`

```c
if (!expand_filepath_with_mode(dir, buf, NULL, 0, CWD_EXPAND )) {
    php_error_docref(NULL, E_WARNING, "Invalid path");
    return 0;
}
```



##### non-thread safe

在非线程安全模式下，流程是完全一样的，最终也会因为无法通过`*`或`?`的检查，抛出`"Invalid path"`



#### 2. recursive=false

接下来看一下`recursive=false`的情况，在这个情况下，线程安全与非线程安全产生了不一样的结果。

`recursive=false`时进入`php_mkdir()`方法，随后进入`php_mkdir_ex()`

![](https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/images/20190320171340.png)

在进行`basedir`检查后进入`VCWD_MKDIR`，这是一个宏命令，在源码中有三处定义，在`php-7.2.16-src/Zend/zend_virtual_cwd.h`中，分别是

- `mkdir(pathname, mode)`
- `php_win32_ioutil_mkdir(pathname, mode)`
- `virtual_mkdir(pathname, mode)`

![](https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/images/20190320172231.png)

注意这三个定义是根据不同的条件执行的，看一下逻辑

```c
#ifdef VIRTUAL_DIR
#define VCWD_MKDIR(pathname, mode) virtual_mkdir(pathname, mode)
#endif

#if defined(ZEND_WIN32)
#define VCWD_MKDIR(pathname, mode) php_win32_ioutil_mkdir(pathname, mode)
#else
#define VCWD_MKDIR(pathname, mode) mkdir(pathname, mode)
```

也就是说，如果定义了`VIRTUAL_DIR`，那么执行的是`virtual_mkdir()`，否则如果是`Windows`系统，就执行`php_win32_ioutil_mkdir()`创建目录，`linux`下则是`mkdir`命令

那么，既然在`recursive=false`的情况下，线程安全与非线程安全出现了不一样的结果，肯定是此处走的分支不一样，一个使用了`virtual_mkdir()`，另一个使用了`php_win32_ioutil_mkdir()`，分别进入两个方法

![](https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/images/20190320173806.png)

在`virtual_mkdir()`中，同上面的情况一样，进行了`virtual_file_ex()`判断，因此也会走到对`*`和`?`的判断，同样因为通不过检查而抛出`"Invalid path"`，而在`php_win32_ioutil_mkdir()`中则是调用了`CreateDirectoryW`创建目录

![](https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/images/20190320174124.png)

`CreateDirectoryW`是`Windows`下创建目录的`API`，走到这个分支并不检查`*`和`?`，因此能够成功创建目录。

> 有关`CreateDirectoryW`参考[Microsoft Doc](https://docs.microsoft.com/en-us/windows/desktop/api/fileapi/nf-fileapi-createdirectoryw)
>
> 与`CreateDirectoryW`对应的还有`CreateDirectoryA`，两个函数功能一样，只是第一个参数的类型不同，一个是`LPCWSTR` 另一个是`LPCSTR` ，这两者是`CHAR` 和`WCHAR`的区别 ，详细可以参考[StackOverflow](https://stackoverflow.com/questions/321413/lpcstr-lpctstr-and-lptstr)

现在剩下最关键的一个问题，什么情况下会走`virtual_mkdir()`的流程，也就是说`VIRTUAL_DIR`是在何处定义的？

![](https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/images/20190320175730.png)

在`php-7.2.16-src/Zend?zend_virtual_cwd.h`line 41定义了这个变量，前置条件是`ZTS`，也就是线程安全的标识，只有在线程安全模式下，才使用`virtual_mkdir()`创建目录，调用的系统函数同样是`CreateDirectoryW`，但是在此之前得先通过`virtual_file_ex()`校验，含有`*`和`?`则无法创建成功。

### 0x04 流程图

![](https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/images/20190321152354.png)

### 0x05 深入

现在清楚了`php`对`mkdir()`的实现，之所以结果不一样是因为`ZTS`与`NTS`下的两种不同的处理流程，那么为什么在`ZTS`模式下，在调用`Windows`的`API`创建目录之前，需要设置一个“虚拟目录”呢？

这里涉及到`php`内核中的`TSRM`机制，也就是`线程安全资源管理器(Thread Safe Resource Manager)` ，这个机制的引入是为了解决线程并发的问题，我们知道，如果线程访问的内存地址空间相同，当一个线程修改资源时会影响其它线程，所以为了确保不会出现资源竞争，`php`将多个资源复制为多份，每个线程需要的资源在当前进程空间中各有一份，各取所取，这样就不会出现竞争问题。

那么不同线程怎么获取自身所需要的资源呢？`php`中通过`ts_allocate_id()`函数实现， 这个函数的作用就是遍历所有线程，为每一个分配一个`线程安全资源id`，每一次调用`ts_allocate_id()`函数时，都会执行这个操作，而为了避免重复分配，这个过程是在调用模块初始化的时候就完成了

`TSRMG`的定义如下，其中`tsrm_get_ls_cache()`有多个定义，但功能是一样的，就是根据资源id的`tls_key`取出相应`value`的过程：

```c
#define TSRMG(id, type, element)	(TSRMG_BULK(id, type)->element)
#define TSRMG_BULK(id, type)	((type) (*((void ***) tsrm_get_ls_cache()))[TSRM_UNSHUFFLE_RSRC_ID(id)])

# define tsrm_tls_get()			pthread_getspecific(tls_key)
```

在启动`cli`或者`cgi`时，都会通过`SAPI`调用`tsrm_startup()`启动`TSRM` ，随后进行模块初始化，在这个过程中分配资源id，初始化时的调用栈如下图所示

![](https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/images/20190321140835.png)

当非ZTS模式时，线程直接调用全局变量的属性， 而ZTS模式设置“虚拟目录”的概念其实就是“根据资源id查找所需的全局变量”的过程，本质上是为了避免线程间资源读取出现竞争，保证了线程安全。

### 0x06 总结

本文通过`php`中`mkdir()`函数在不同环境下表现结果不一致的现象，分析了`php`内核对`mkdir()`函数的实现，引申出`php`中线程安全与非线程安全两个重要的机制，抛砖引玉，如有表述不妥或者错误之处欢迎指正，最后感谢@maple提出最初的问题以及探讨过程中给予的莫大的帮助。



参考：

http://php.net/manual/en/function.mkdir.php

http://blog.codinglabs.org/articles/zend-thread-safety.html

https://segmentfault.com/a/1190000010004035



<script>pangu.spacingPage();</script>





