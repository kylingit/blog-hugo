---
title: 基于Python的简单验证码识别程序
date: 2016-06-05 22:59:18
tags: Python
categories: Coding
---
<script src="https://ob5vt1k7f.qnssl.com/pangu.js"></script>

## 主体思想

     1. 批量获取验证码 
     2. 对图片去噪，二值化 
     3. 对图片进行切割，获取单个字符
     4. 选取清晰的字符作为标准库
     5. 待识别验证码经过处理后与标准库进行逐像素比较，选取最相近的组合作为识别结果为图片命名

<!-- more -->

## 函数介绍

    download()      批量获取验证码  
    binary()        图像的去噪和二值化处理  
    division()      把验证码图片按单个字符切割开(关键)  
    recognize()     验证码识别(关键)

## 项目地址
[Github](https://github.com/kylingit/Captcha_recognize)

## 代码参考

```python
#!/usr/bin/env python
# -*- coding: UTF-8 -*-

import os, Image, time
import urllib, random

pic_path = "D:/code/pic/"               #下载保存的路径
result_path = "D:/code/result/"         #识别后保存的路径
font_path = "D:/code/font/"             #去噪和二值化后保存的路径
standard_path = "D:/code/standard/"     #标准字符库路径
fonts_path = "D:/code/fonts/"           #图片切割后保存的路径

##批量下载验证码,用随机数命名##
def download(path):
    for i in range(50):
        url = 'http://system.ruanko.com/validateImage.jsp'
        print "download", i
        file(path + "%04d.jpg" % random.randrange(10000), "wb").write(urllib.urlopen(url).read())
        time.sleep(0.1)
    return path

##图像的去噪和二值化处理##
def binary(pic_f, saved_f):
    img = Image.open(pic_f)
    img = img.convert("RGBA")  
    pixdata = img.load()
    for y in xrange(img.size[1]):
        for x in xrange(img.size[0]):
            if pixdata[x, y][0] < 90:
                pixdata[x, y] = (0, 0, 0, 255)
    for y in xrange(img.size[1]):
        for x in xrange(img.size[0]):
            if pixdata[x, y][1] < 136:
                pixdata[x, y] = (0, 0, 0, 255)
    for y in xrange(img.size[1]):
        for x in xrange(img.size[0]):
            if pixdata[x, y][2] > 0:
                pixdata[x, y] = (255, 255, 255, 255)
    img.save(saved_f, "png")
    return img

nume = 0
##把验证码图片按单个字符切割开##
def division(img):
    global nume
    font = []
    (Width, Height) = img.size
    pix = img.load()
    x0 = []
    y0 = []
    for x in range(0, Width):
        pix_0 = 0
        for y in range(0, Height):
            if pix[x, y] == 0:                      #遍历每一列像素点为0的个数,若某一列像素点全为0而下一列存在不为0的点,则可认为此处为边界
                pix_0 += 1
        y0.append(pix_0)
        if pix_0 > 0:
            x0.append(x)
    preWidth = []
    for i in range(4):
        for j in range(1, Width):
            if (y0[j] != 0) & (y0[j+1] != 0):
                preWidth.append(j+1)                #连续非0的个数即为分割后的宽度preWidth
                break
    for i in range(4):
        x = i*13 + 7                                #模板的长*宽需要微调
        y = 3
        temp = img.crop((x, y, x+preWidth[i]+1, 16))#切割宽度+1后结果比较精确
        temp.save(fonts_path +" %d.png" % nume)
        nume = nume + 1
        font.append(temp)
    return font

##分隔出来的字符与预先定义的标准字符库中的结果逐个像素进行对比找出差别最小的项##
def recognize(img):
    fontMods = []
    for i in range(0, 10):  
        fontMods.append((str(i), Image.open(standard_path + "%d.png" % i)))
	#此句针对全数字的验证码，按数字值对单个字符命名并保存
        #fontMods.append((str(i), Image.open(standard_path +"%02d.bmp" % ord('1'))))
    #for i in range(65, 91):
	#以下针对数字+大小写字母的验证码，按ASCII码值对单个字符命名并保存
        #c = chr(i) 
        #fontMods.append((c, Image.open(standard_path +"%s.bmp" % ord('A'))))
    #for i in range(97, 123):  
        #s = chr(i)
        #fontMods.append((s, Image.open(standard_path +"%s.bmp" % ord('a'))))
    result = ""
    img = img.convert("1")
    font = division(img)
    for i in font:
        target = i                                  #标准字符库
        points = []
        #取出验证码并分割后与标准字符库进行逐像素比较
        for mod in fontMods:                        
            diffs = 0
            for yi in range(10):
                for xi in range(7):
                    if mod[1].getpixel((xi, yi)) != target.getpixel((xi, yi)):  
                        diffs += 1
            points.append((diffs, mod[0]))
        points.sort()
        result += points[0][1]
    return result

if __name__ == '__main__':
    codedir = download(pic_path)                        
    #批量下载验证码图片
    for imgfile in os.listdir(codedir):
        if imgfile.endswith(".jpg"):
            #识别后路径
            result = result_path                            
            #去噪和二值化
            img = binary(pic_path + imgfile, font_path + imgfile)
            #识别
            num = recognize(img)                                   
            result += (num + ".png")
            print "save to", result
            img.save(result)
```


  [1]: https://github.com/kylingit/Captcha_recognize

<script>pangu.spacingPage();</script>