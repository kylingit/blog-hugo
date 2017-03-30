---
title: URP教务系统教学评价Python脚本
date: 2016-06-08 23:14:22
tags: Python
categories: Coding
---
<script src="https://ob5vt1k7f.qnssl.com/pangu.js"></script>

## 简述
夏天到了，又到了 ~~繁殖~~ 评课的季节→_→

URP评课程序2.0，针对第二学期的情况作了一些修改，又可以欢乐地一键评课了~

针对URP教务系统教学评价的python脚本，本地验证码登录，python版本要求3.4，自行更改相应教务处网站地址，运行
`python3 URP_instructional_evaluation_v2.0.py`

## 项目地址 
[Github](https://github.com/kylingit/URP_instructional_evaluation)

<!-- more -->
## 部分代码

```
##登录##
def login(user, password, code):

##获取cookie##
def setCookie():

##提取验证码(手动输入验证码)##
def getVerify():

##获取选课信息##
def getInfo():

##提交评课信息##
def postPj(br, pr, bm, pm):
```
```
##获取cookie##
def setCookie():
    cookie = http.cookiejar.CookieJar() 
    cookieProc = urllib.request.HTTPCookieProcessor(cookie) 
    opener = urllib.request.build_opener(cookieProc) 
    urllib.request.install_opener(opener)
```
```
##提取验证码(手动输入验证码)##
def getVerify():
    setCookie()
    vrifycodeUrl = 'http://xxx.edu.cn/validateCodeAction.do'
    file = urllib.request.urlopen(vrifycodeUrl)
    pic= file.read()
    #path = '/home/username/code.jpg'
    path = 'D:/code.jpg'
    try:
        localpic = open(path, 'wb')
        localpic.write(pic)
        localpic.close()
        print ('获取验证码成功,%s.' %path)
    except IOError:
        print ('获取验证码失败,请重新运行程序.')
    code = input("验证码: ")
    return code
```
```
##模拟登录##
url = 'http://xxx.edu.cn/loginAction.do'         ##登录地址
header = {}
    header['Accept'] = 'text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8'
    header['Accept-Encoding'] = 'gzip, deflate'
    header['Accept-Language'] = 'zh-CN,zh;q=0.8,en-US;q=0.5,en;q=0.3'
    header['Connection'] = 'keep-alive'
    header['Host'] = 'xxx.edu.cn'
    header['Referer'] = 'http://xxx.edu.cn/loginAction.do'
    header['User-Agent'] = 'Mozilla/5.0 (Windows NT 6.3; WOW64; rv:39.0) Gecko/20100101 Firefox/39.0'
    data = urllib.parse.urlencode(data).encode('gb2312')
    req = urllib.request.Request(url,data,header)
    response = urllib.request.urlopen(req)
    html = response.read().decode('gb2312')
```
```
##提交评价数据包##
data = urllib.parse.urlencode(data).encode('gb2312')
    pjurl = 'http://xxx.edu.cn/jxpgXsAction.do?oper=wjpg'            ##提交评价结果页面
    pjreq = urllib.request.Request(pjurl,data,pjheader)
    pjresponse = urllib.request.urlopen(pjreq)
    pjhtml = pjresponse.read().decode('gb2312')
```

<script>pangu.spacingPage();</script>
