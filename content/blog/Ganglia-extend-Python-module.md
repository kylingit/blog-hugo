---
title: Ganglia 扩展 Python模块
date: 2016-07-30 14:32:18
tags: Ganglia
categories: Coding
---
<script src="https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/pangu.js"></script>

Ganglia支持很多模块的扩展，这里介绍python模块的扩展

### 环境需求
- Ganglia 3.1.x
- Python 2.5+
- Python开发头文件

<!-- more -->

### 配置
以下两个目录必须存在，没有则创建
`/usr/lib/ganglia/python_modules/`
`/etc/ganglia/conf.d/`

#### 修改配置文件
 1. `vim /etc/ganglia/gmond.conf`
在`modules{}`中添加
```
modules {
  module {
    name = "python_module"
    path = "/usr/lib/ganglia/modpython.so"
    params = "/usr/lib/ganglia/python_modules"
  }
  module {
    #other modules
  }
```
 2. 添加`include('/etc/ganglia/conf.d/*.pyconf')`
```
include ('/etc/ganglia/conf.d/*.conf')
include ('/etc/ganglia/conf.d/*.pyconf')
```
  
#### 放置 .py 及 .pyconf
 1. 在`/usr/lib/ganglia/python_modules/`下放置python模块代码(.py)
```
root@ubuntu:~# ls /usr/lib/ganglia/python_modules/
process_count.py
```
 2. 在`/etc/ganglia/conf.d/`下放置python模块配置文件(.pyconf)
```
root@ubuntu:~# ls /etc/ganglia/conf.d/
es_syslog-ng.pyconf
```
重启服务就可以了
`service ganglia-monitor restart`

### python模块模板
模块中必须包含以下的三个方法
```
def metric_init(params):
def metric_cleanup():
def metric_handler(name):
```
前面两个方法的名字必须是一定的，最后一个`metric_handler`可以任意命名，具体可以看example
```
import random
descriptors = list()
Random_Max = 50
Constant_Value = 50

def Random_Numbers(name):
    '''Return a random number.'''
    global Random_Max
    return int(random.uniform(0,Random_Max))

def Constant_Number(name):
    '''Return a constant number.'''
    global Constant_Value
    return int(Constant_Value)

def metric_init(params):
    '''Initialize the random number generator and create the
    metric definition dictionary object for each metric.'''
    global descriptors
    global Random_Max
    global Constant_Value
    random.seed()

    print '[pyexample] Received the following parameters'
    print params

    if 'RandomMax' in params:
        Random_Max = int(params['RandomMax'])
    if 'ConstantValue' in params:
        Constant_Value = int(params['ConstantValue'])

    d1 = {'name': 'PyRandom_Numbers',
        'call_back': Random_Numbers,
        'time_max': 90,
        'value_type': 'uint',
        'units': 'N',
        'slope': 'both',
        'format': '%u',
        'description': 'Example module metric (random numbers)',
        'groups': 'example,random'}

    d2 = {'name': 'PyConstant_Number',
        'call_back': Constant_Number,
        'time_max': 90,
        'value_type': 'uint',
        'units': 'N',
        'slope': 'zero',
        'format': '%hu',
        'description': 'Example module constant (constant number)'}

    descriptors = [d1,d2]
    return descriptors

def metric_cleanup():
    '''Clean up the metric module.'''
    pass

#This code is for debugging and unit testing    
if __name__ == '__main__':
    params = {'RandomMax': '500',
        'ConstantValue': '322'}
    metric_init(params)
    for d in descriptors:
        v = d['call_back'](d['name'])
        print 'value for %s is %u' % (d['name'],  v)
```

<script>pangu.spacingPage();</script>