---
title: Python2.7 中 UnicodeEncodeError:'ascii' codec can't encode characters 异常解决
date: 2016-08-15 14:32:18
tags: Python
categories: Coding
---
<script src="https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/pangu.js"></script>

Python的编码问题一直是一个它的一个缺点，特别是在处理中文上。Python提供了Unicode, str, utf-8, ascii等编码的相互转换，然而还是烦琐易错。
进行sqlite3数据读取并存入文件时碰到了错误 :
`UnicodeEncodeError: 'ascii' codec can't encode characters in position 19-22: ordinal not in range(128)`

原因是数据库中含有中文字段，Unicode编码与ASCII编码的不兼容，这个Python脚本文件是由UTF-8编码的，同时Sqlite3数据库存取的也是UTF-8格式，而Python默认环境编码是Ascii:
```
>>> import sys
>>> print sys.getdefaultencoding()
ascii
```

Python调用ascii编码解码程序去处理字符流，当字符流不属于ascii范围内，就会抛出异常`ordinal not in range(128)`，解决方法有三种

<!-- more -->

 - 方法一

更改默认编码
```
import sys
reload(sys)
sys.setdefaultencoding('utf-8')
```

把这段代码加在Python文件头部，即可解决异常。

 - 方法二

在打开文件时指定编码

```
import codecs
fp = codecs.open('output.txt', 'a', 'utf-8')
fp.write(data)
fp.close()
```
 - 方法三

直接用系统输出byte，不用print

`sys.stdout.buffer.write(data)`

或者

`os.write(sys.stdout.fileno(), data)`

<script>pangu.spacingPage();</script>