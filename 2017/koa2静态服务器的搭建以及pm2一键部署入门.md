>使用某一技术栈多了，自然而然就要考虑属于自己常用的脚手架了，毕竟搬砖也要自己总结一套最有效率的搬砖方法。

本人目前使用较多的是react方面的技术栈，react官方推荐使用的是[create-react-app](https://github.com/facebookincubator/create-react-app),本文不做过多讲解。该脚手架将webpack隐藏了起来，使用者不必去关注具体的webpack实现细节，而是操作相关的脚本即可。(当然，作为一名优秀的程序员，就算没有直接接触webpack，你也需要去官网看看create-react-app相关的文档)

此脚手架是基于create-react-app搭建了基于react+redux的脚手架，加上了基于koa2(也有express版本)实现的静态服务器，并且配置了pm2的自动化部署。
项目目录如下：
```bash
create-app-starter
├── README.md
├── node_modules
├── package.json
├── ecosystem.json  //pm2配置文件
├── .gitignore
├── public
│   └── favicon.ico
│   └── index.html
│   └── manifest.json
├── src
|   └── components #表现层组件
|   |   └── Loading
|   |       ├── index.js
|   |       └── index.scss
|   |   └── Dialog
|   └── containers #最外层的容器层，主要是provider包裹，传递store树
|   └── layout  #主要是整个容器样式进行初始化，在这可以添加nav,header,footer等无状态组件
|   └── routes   #路由层，主要通过路由来划分不同的组件
|   |   └── index.js  #路由的设置，推荐使用react-router
|   |   └── Home
|   |       ├── components # 容器型组件
|   |       ├── containers # 主要是使用connect 连接组件
|   |       ├── modules #定义action，并且执行reducers
|   |       └── index.js #结合webpack，根据路由做按需加载的操作 
|   └── store
|   |   ├── createStore.js  #创建store,添加一些middlewares
|   |   ├── location.js
|   |   └── reducers.js  //封装reducer，结合combineReducer  
|   ├── styles 
|   ├── registerServiceWorker.js  #create-react-app自带的servicework
|   └── index.js #入口文件
└── server #静态服务器
```
关于脚手架具体的说明，请各位去我的[github](https://github.com/PerkinJ/create-app-starter)上查看

下面开始介绍一下koa2搭建静态服务器。我们需要启动我们的客户端，静态服务器，无非就是读取静态资源，koa2的最大特点就是使用async/await来实现异步。
```bash
#安装
npm install koa

#安装koa2相关的中间件
npm install koa-static koa-morgan --save
```

代码如下：
 
app.js
```js
const Koa = require('koa')
const morgan = require('koa-morgan')
const path = require('path');
const static = require('koa-static')
const fs = require('fs')

const app = new Koa();

// logger
app.use(morgan(':remote-addr - :remote-user [:date[clf]] ":method :url HTTP/:http-version" :status :res[content-length] :response-time ms'));

// static assets
app.use(static(path.resolve(__dirname, '..', 'build')));

//异步读取文件的形式
app.use(async (ctx,next) =>{
	ctx.type = 'html';
	ctx.body = await fs.createReadStream(path.resolve(__dirname, '..', 'build', 'index.html'));

})


module.exports = app;
```

index.js
```js
'use strict';

const app = require('./app');

const PORT = process.env.PORT || 9000;

app.listen(PORT, () => {
  console.log(`App listening on port ${PORT}!`);
});
```
此时运行
```bash
node server
```
即可，当然也可以自定义端口
```bash
PORT=8000 node server
```
既然实现了静态服务器，当我们需要自己部署项目时，怎么去通过服务器部署自己的项目呢。

> 当然部署到服务器的方法有很多。仅仅node而言，pm2,forever等，甚至anywhere这样的，也能成功。或者粗暴的把整个项目cp到服务器上，再运行，都能访问。本文不讨论何种方法好。

当你的项目每次都要改动时，本地改完，push到仓库内，再从服务器pull下来，重新build，再采用如上的各种方法。这样无形之中就加大了时间成本，因而我的解决方法时采用pm2的自动化部署方法，来实现本地一键部署到服务器。

关于pm2的其他功能，本文不做相信处理，可以查看[pm2官方文档](http://pm2.keymetrics.io/docs/usage/cluster-mode/)。

实现一键部署，我们需要做如下操作：
1. 本地跟服务器都需要安装node,git,pm2
```bash
npm install pm2 -g
```
2. 配置SSH
    1. 在Github上添加Deploy Keys
    ```bash
    ssh-keygen -t rsa
    cat ~/.ssh/id_rsa.pub
    ```
    然后复制内容，添加到Github上对应的项目仓库Settings下的Deploy keys中
    2.将 .pub 文件复制到服务器的 .ssh 目录， 并 cat
    ```bash
     # 将本地key添加到服务器的authorized_keys中
    id_dsa.pub >> ~/.ssh/authorized_keys
    ```
    3. 本地自动登录到服务器,避免每次都输入密码
    ```
    # name 是你的服务器用户名 server 是你服务器的地址
    
    ssh-copy-id name@serer
    ```
3. 配置pm2
    1. 在服务器根目录下创建/www/website
    ```bash
    sudo mkdir /www
    cd /www
    sudo mkdir website
    #修改权限,先切换到www
    
    sudo chmod 777 website
    ```
    2. 下面是脚手架的ecosystem.json
    
    ```js
    {
	"apps":{
		"name":"**",
		"script":"/server/index.js",
		"env":{
			"COMMON_VARIABLE":"true"
		},
		"env_production":{
			"NODE_ENV":"production"
		}
	},
	"deploy":{
		"production":{
			"user":"***", //服务器名称
			"host":["192.168.2.31"], //服务器ip
			"ref":"origin/master",
			"repo":"",
			"path":"/www/website/production", //项目目录
			"ssh_options":"StrictHostKeyChecking=no",
			"post-deploy":"npm install && npm run build && pm2 startOrRestart ecosystem.json --env production",
			"env":{
				"NODE_ENV":"production"
		    	}
	    	}
    	}
    }
    ```
    3. 再服务器根目录编辑bash脚本，允许pm2脚本执行
    ```
    vi .bashrc
    ```
    注释掉
    ```
    #CASE
    # ...
    ```
    再
    ```
    source .bashrc
    ```
    4. 执行
    ```bash 
    pm2 deploy ecosystem.json production setup
    ```
    此时会在服务器 /www/website/production新建三个目录
    
    5. 发布
    ```
    pm2 deploy ecosystem.json production
    ```
    不出意外，即发布成功。
    
按照上述步骤操作，就可以实现利用pm2本地部署项目到服务器，这样就省掉了登入服务器取更新代码的步骤了。

>当然，pm2还结合Nginx一同服务，具体的Nginx配置，请参考我另外一篇文章[《学习Nginx配置》](http://perkinzone.cn/2017/07/16/%E5%AD%A6%E4%B9%A0%E7%9B%B8%E5%85%B3Nginx%E9%85%8D%E7%BD%AE/)