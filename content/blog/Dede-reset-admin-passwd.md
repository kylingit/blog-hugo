---
title: DedeCMS V5.7 SP2 第二处缺陷可重置管理员密码
date: 2018-01-19 15:20:32
tags: [vul,sec,dedecms]
categories: Security
---
<script src="https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/pangu.js"></script>

### 0x01 概述
上回分析的dede重置密码漏洞有一定局限性，一是只能影响没有设置密保问题的用户，二是不能重置管理员admin的密码，原因当时也说了，管理员信息存在另一个表`dede_admin`中，而且管理员默认不允许从前台登录，所以就算更改了`dede_member`里`admin`的密码也没法登录。但是前几天又有一个缺陷被爆出来，可以绕过一些判断条件从而从前台登录管理员账户，配合上一个重置密码漏洞，可以达到从前台修改`dede_admin`表里是密码，也就是真正修改了管理员密码。

下面来简单分析一下

### 0x02 漏洞分析
先来看一下DedeCMS判断登录用户的逻辑
`include/memberlogin.class.php:292`
```
function IsLogin()
{
    if($this->M_ID > 0) return TRUE;
    else return FALSE;
}
```
跟进`$this->M_ID`看一下，170行
```
$this->M_ID = $this->GetNum(GetCookie("DedeUserID"));
```
GetNum()`include/memberlogin.class.php:398`
```
/**
*  获取整数值
*
* @access    public
* @param     string  $fnum  处理的数值
* @return    string
*/
function GetNum($fnum){
    $fnum = preg_replace("/[^0-9\.]/", '', $fnum);
    return $fnum;
}
```
正则匹配，去除了数字以外的字符，这里就可以构造一个利用点，一会儿再看

看一下`GetCookie()`
`include/helpers/cookie.helper.php:54`

![GetCookie](https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/images/IO7Rf)

关键点在这个判断条件
```
if($_COOKIE[$key.'__ckMd5'] != substr(md5($cfg_cookie_encode.$_COOKIE[$key]),0,16))
```
就是说从cookie中取到`DedeUserID__ckMd5`值，与`md5($cfg_cookie_encode.$_COOKIE[$key])`取前16位比较，相等才能进行下一步

我们知道admin的`DedeUserID`为1，现在需要知道`DedeUserID__ckMd5`的值

其实再思考一下，就算我们不知道admin的`DedeUserID__ckMd5`值，只要能过这个if条件就能绕过接着往下走了，那我们可不可以利用其他用户来绕过if条件呢？

在本程序中从数据库取用户的过程其实很简单，就是简单的查询语句`Select * From #@__member where mid='$mid'`。当我们利用其他用户的cookie通过了上面的if判断，然后修改mid为admin的id(1)，就可以从前台登录到admin账户。
那么如何在请求过程中修改`DedeUserID`的值让它能和admin的id相等呢？

#### 利用点一
我们使进入`GetNum`方法的参数为`数字1+字母`的形式，经过正则替换就会变成`1`，也就是`$this->M_ID`的值，然后带入数据库查询

![GetNum](https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/images/4zlB8)
`$fnum`为`1qqqq`的情况，经过正则替换后值成为了1

#### 利用点二
在`include/memberlogin.class.php:178`有这么一行代码
```
$this->M_ID = intval($this->M_ID);
```
对`$this->M_ID`进行了整数类型转换，假设注册一个用户名，经过`intval`转换后为`1`就能使查询条件变成`Select * From #@__member where mid='1'`，也就取出了管理员在`dede_member`表里的密码，此时配合上一个漏洞，我们已经修改了`dede_member`中管理员的密码，只要在前台再进行一次修改密码操作，就能真正修改admin的密码。

![intval](https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/images/Av52c)
这是调试的时候注册用户名为`0000001`的情况，经过`intval`转换后`M_ID`的值变成了1

下面看一下如何从前台登录admin账户

在`index.php`里有一个`最近访客记录`的功能，
```
else
{
    require_once(DEDEMEMBER.'/inc/config_space.php');
    if($action == '')
    {
        include_once(DEDEINC."/channelunit.func.php");
        $dpl = new DedeTemplate();
        $tplfile = DEDEMEMBER."/space/{$_vars['spacestyle']}/index.htm";

        //更新最近访客记录及站点统计记录
        $vtime = time();
        $last_vtime = GetCookie('last_vtime');
        $last_vid = GetCookie('last_vid');
        if(empty($last_vtime))
        {
            $last_vtime = 0;
        }
        if($vtime - $last_vtime > 3600 || !preg_match('#,'.$uid.',#i', ','.$last_vid.',') )
        {
            if($last_vid!='')
            {
                $last_vids = explode(',',$last_vid);
                $i = 0;
                $last_vid = $uid;
                foreach($last_vids as $lsid)
                {
                    if($i>10)
                    {
                        break;
                    }
                    else if($lsid != $uid)
                    {
                        $i++;
                        $last_vid .= ','.$last_vid;
                    }
                }
            }
            else
            {
                $last_vid = $uid;
            }
            PutCookie('last_vtime', $vtime, 3600*24, '/');
            PutCookie('last_vid', $last_vid, 3600*24, '/');
```
else条件是当访问页面`http://127.0.0.1/dedecms/uploads/member/index.php?uid=1111`传入的uid不为空时进入

当我们传入的`last_vid`为空的时候，`$last_vid = $uid;`而`uid`是我们能控制的，所以我们就能控制传给`PutCookie`的参数，进入`PutCookie`方法
```
if ( ! function_exists('PutCookie'))
{
    function PutCookie($key, $value, $kptime=0, $pa="/")
    {
        global $cfg_cookie_encode,$cfg_domain_cookie;
        setcookie($key, $value, time()+$kptime, $pa,$cfg_domain_cookie);
        setcookie($key.'__ckMd5', substr(md5($cfg_cookie_encode.$value),0,16), time()+$kptime, $pa,$cfg_domain_cookie);
    }
}
```
在这里设置了`last_vid__ckMd5`的值

所以攻击流程已经明确了

- 注册一个普通用户，用户名满足`数字1+字母`的形式，或者经过`intval()`后值为`1`
- 访问用户主页，记录cookie中`last_vid__ckMd5`的值
- 访问index页面，替换cookie中`DedeUserID`和`DedeUserID__ckMd5`的值，替换成我们注册的用户名和`last_vid__ckMd5`，就能登录到前台admin


### 0x03 漏洞利用
1. 前台注册普通用户，这里注册一个`1qqqq`
2. 访问`/member/index.php?uid=1qqqq`，获取`last_vid__ckMd5`的值
	![uid](https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/images/2izw4)
3. 访问`/member/index.php`，替换`DedeUserID`和`DedeUserID__ckMd5`的值
	![admin](https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/images/3uAj1)
	可以发现以admin身份成功登录到了前台
    ![admin](https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/images/aGGc5)
4. 同样的，修改密码访问`member/edit_baseinfo.php`，还是要修改cookie值
	![reset passwd](https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/images/OacVb)
	原登录密码就是我们利用上一个漏洞修改的密码，也就是`dede_member`表中的admin密码，这样就达到了真正修改admin的密码
	![reset admin passwd](https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/images/OUrvR)
	更新数据库的时候判断如果是管理员，就更新admin表中的数据

### 0x04 总结
还是判断不够严谨，这回有两处可导致判断条件的绕过，有时候一个漏洞影响力有限的时候也不能轻视，往往配合另一处缺陷就可以造成很大的危害

参考：
- https://xianzhi.aliyun.com/forum/topic/1961
- https://xianzhi.aliyun.com/forum/topic/1959


<script>pangu.spacingPage();</script>
