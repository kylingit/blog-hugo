---
title: DedeCMS V5.7 SP2前台任意用户密码重置漏洞分析
date: 2018-01-11 14:12:17
tags: [vul,sec,dedecms]
categories: Security
---
<script src="https://ob5vt1k7f.qnssl.com/pangu.js"></script>

#### 0x01 概述
DEDECMS在2018-01-09更新了V5.7 SP2正式版，然后在[seebug](https://www.seebug.org/vuldb/ssvid-97074)有人提交存在前台任意用户密码修改漏洞。下面简单分析一下。

#### 0x02 影响版本
2018-01-09及之前的版本

#### 0x03 漏洞分析
问题出现在`member/resetpassword.php`75行:
```php
else if($dopost == "safequestion")
{
    $mid = preg_replace("#[^0-9]#", "", $id);
    $sql = "SELECT safequestion,safeanswer,userid,email FROM #@__member WHERE mid = '$mid'";
    $row = $db->GetOne($sql);
    if(empty($safequestion)) $safequestion = '';

    if(empty($safeanswer)) $safeanswer = '';

    if($row['safequestion'] == $safequestion && $row['safeanswer'] == $safeanswer)
    {
        sn($mid, $row['userid'], $row['email'], 'N');
        exit();
    }
    else
    {
        ShowMsg("对不起，您的安全问题或答案回答错误","-1");
        exit();
    }
}
```
重置密码的时候需要进入`sn`函数，在这之前进行if判断`if($row['safequestion'] == $safequestion && $row['safeanswer'] == $safeanswer)`
当用户没有设置安全问题和答案时`$row['safeanswer']`为`0`，后面一个条件成立，所以只要前面`$row['safequestion'] == $safequestion`成立就可以进入`sn`函数

此时默认的`$row['safequestion']`即为`0`，我们可以控制的变量是`$safequestion`，在此之前还需经过`if(empty($safequestion)) $safequestion = '';`判断，如果这个if成立即当`$safequestion = ''`时就不能通过前半个if判断了，所以我们要让`$safequestion`不为空而且让`'0' == $safequestion`成立

下面来看php中弱类型转换问题

![php](https://ob5vt1k7f.qnssl.com/edFq9)

可以看到当我们传进`0.0`时，`empty($safequestion)`就不成立了，而`$row['safequestion'] == $safequestion`即`'0' == '0.0'`成立，所以可以进入`sn`方法。除了`'0.0'`，`'0.'` `'0e123'`等都可以绕过这个判断，因为`0en`被认为是0的n次方

跟进`sn`方法

![sn](https://ob5vt1k7f.qnssl.com/xZxqc)

从数据库取出一个临时密码`SELECT * FROM #@__pwd_tmp WHERE mid = '$mid'`，这里的`mid`我们可以控制，如果用户存在，发送含有临时密码的邮件，并且有个10分钟的限制(这里为了调试方便我把时间缩短了)

跟进`newmail`函数

![newmail](https://ob5vt1k7f.qnssl.com/0nL4H)

可以看到`$randval`是一个8位随机字符串，而且先进行了md5再插入到数据库，理论上我们不好破解，但是注意85行和98行的`return ShowMsg('稍后跳转到修改页', $cfg_basehost . $cfg_memberurl . "/resetpassword.php?dopost=getpasswd&amp;id=" . $mid . "&amp;key=" . $randval);`，把含有`$randval`的链接直接返回显示在页面上，所以这里就没有必要去猜这个临时密码。有了这个临时密码就可以重置任意用户的密码。


#### 0x04 漏洞利用
我们先注册一个用户，然后构造一个请求，`GET /dedecms/uploads/member/resetpassword.php?i=0.0&dopost=safequestion&safequestion=0e123&safeanswer=&id=1`，发送后可以看到页面跳转，然后返回含有key的链接，

![key](https://ob5vt1k7f.qnssl.com/eSNsc)
利用这个key可以进入重置密码流程，简单看一下

重置密码`/member/resetpassword.php?dopost=getpasswd&id=5`

![resetpwd1](https://ob5vt1k7f.qnssl.com/oV7zo)
先从`dede_pwd_tmp`表取出`mid`为5的临时密码

![resetpwd2](https://ob5vt1k7f.qnssl.com/aIWLU)
与传入的临时密码MD5比较，通过验证就更新用户表`dede_member`为新的密码，同时删除临时密码

![resetpwd3](https://ob5vt1k7f.qnssl.com/J1mks)
![newpwd](https://ob5vt1k7f.qnssl.com/EMzcr)

这样我们就可以重置任意用户的密码了——除了管理员，因为管理员信息存在另一个表`dede_admin`中，而且管理员默认不允许从前台登录，所以就算更改了`dede_member`里`admin`的密码也没法登录。


#### 0x05 总结
总的来说这个漏洞不算复杂，关键点就是php弱类型安全问题，这个已经有很多案例了，同时页面跳转的过程中泄露了临时的key，实际中一个尽量避免这种关键的参数泄露。

参考：https://www.seebug.org/vuldb/ssvid-97074

<script>pangu.spacingPage();</script>