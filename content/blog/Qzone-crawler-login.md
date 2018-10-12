---
title: QQ 空间爬虫之模拟登录
date: 2017-03-24 14:48:15
tags: [Python, Mysql, Qzone, crawler]
categories: Coding
---

<script src="https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/pangu.js"></script>

想要抓取 QQ 空间数据的第一步就是登录空间，通过好友关系获取说说，日志，留言等。

话说 QQ 空间登录算法好变态...4000+ 行 [js](https://qzonestyle.gtimg.cn/c/=/qzone/v8/engine/migrate-plugin.js,/qzone/v8/engine/console-plus/console-plus.js,/qzone/v8/engine/request/request_61221.js,/qzone/v8/core/interface_mini.js) 加密，想要读懂该算法也是需要耗费大段时间，好在 github 上有大神实现了该算法，感谢 [gera2ld](https://github.com/gera2ld) 大神提供的登录库，为我们省去了大量时间，详情戳 [qqlib](https://github.com/gera2ld/qqlib)

关于 QQ 空间具体是如何登录的，分析起来比较复杂，关联的 url 也比较多，需要处理的参数更多，如果需要的话会单独拿出来分析，这里跟我们的项目关系不是很大，我们只要能够登录上并且保持登录状态就可以了，所以偷个懒...

可以直接用`pip`安装`qqlib`, 然后`import qqlib`使用该库，但由于`qqlib`更新频繁，怕到后来有些不兼容，这里选用 2017-03-04 更新的版本，自己加了几个方法的实现。

本爬虫一个特点就是可以利用上次登录的 cookies 登录，不必每次都通过账号密码登录，当然第一次登录还是要通过账号密码认证，之后从保存的 cookies文件获取内容。cookies 有一定有效期，读取之前会判断该 cookies 是否失效。

<!-- more -->

#### 1. 登录流程
![login](https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/images/vrtFN)

#### 2. 常规登录
这段是`qqlib`的示例，可以处理含验证码的登录
```
def login(self):
	exc = None
	while True:
		try:
			if exc is None:
				self.qq.login()
				break
			else:
				verifier = exc.verifier
				open('verify.jpg', 'wb').write(verifier.fetch_image())
				print('saved verify.jpg')
				vcode = input('input verify:')
				verifier.verify(vcode)
				exc = None
		except qqlib.NeedVerifyCode as e:
			if e.message != None:
				print e.message
			exc = e
```
#### 3. 从 cookies 登录
##### 3.1 保存 cookies
登录成功后将 cookies 保存下来，以便下次直接从文件中获取 cookies 用以认证，省去每次从账号密码登录的繁琐，同时也能防止检测到频繁登录(虽然并没有什么用...)
利用 `requests` 库的 [dict_from_cookiejar()](http://docs.python-requests.org/zh_CN/latest/api.html#requests.utils.dict_from_cookiejar) 方法可以将 `cookiejar` 对象转换为字典，然后利用 `pickle` 模块的 [dump()](https://docs.python.org/2/library/pickle.html#pickle.dump) 方法将对象存储在文件中
```
def save_cookie_to_file(cookie, cookie_file):
	with open(cookie_file, 'w') as f:
		pickle.dump(requests.utils.dict_from_cookiejar(cookie), f)
```

##### 3.2 读取 cookies
读取 cookies 方法和保存时一样，只不过把上面的方法反过来执行，利用 [cookiejar_from_dict()](http://docs.python-requests.org/zh_CN/latest/api.html#requests.utils.cookiejar_from_dict) 和 [load()](https://docs.python.org/2/library/pickle.html#pickle.load) 方法
```
def load_cookie_from_file(cookie_file):
	if os.path.isfile(cookie_file):
		with open(cookie_file) as f:
			cookie = requests.utils.cookiejar_from_dict(pickle.load(f))
			return cookie
	return None
```
##### 3.3 `cookiejar` 对象转字符串
由于 cookies 直接附带在 Headers 中一起发给服务器，所以要将 `cookiejar` 对象转成字符串，和其他字段一起组成 Headers
```
def cookiejar_to_string(cookies):
	if cookies == None:
		return None
	else:
		cookie = ''
		for keys, values in cookies.iteritems():
			cookie += keys+ '=' + values + ';'
		cookie = cookie[:len(cookie)-1]
		return cookie
```

#### 4. `g_tk` 值
不管是直接登录还是从 cookies 登录，非常重要的一点是为了获取 `p_skey` 或 `skey` 值，这两个值用来计算 `g_tk` 值，计算方法已经有代码能够实现了
```
def g_tk(self):
	h = 5381
	cookies = self.session.cookies
	s = cookies.get('p_skey') or cookies.get('skey') or ''
	for c in s:
		h += (h << 5) + ord(c)
	return h & 0x7fffffff
```

#### 5. 检查登录
检查是否登录成功思想就是访问该 qq 的用户资料界面，如果能获取成功说明模拟登录成功
该请求是这样子的
```
https://h5.qzone.qq.com/proxy/domain/r.qzone.qq.com/cgi-bin/user/cgi_personal_card?uin=用户qq&g_tk=g_tk值
```
请求成功返回一段 json，如果 `g_tk` 值错误或者请求不合法的话返回错误码 403
![user info](https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/images/jQpGH)

#### 6. 后续
这样我们有了可用的 cookies ，从 cookies 计算`g_tk`值，有了`g_tk`和好友 qq 号就可以拼接 url 批量获取好友数据了~

<script>pangu.spacingPage();</script>








