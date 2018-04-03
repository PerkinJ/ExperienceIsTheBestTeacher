# 前端Docker化:(二)Nginx容器部署实践

> 最近本人在学docker，虽然网上关于docker的文档或者资料很多，但针对于前端开发人员的或者前端实践的文章并不多，故总结了这一系列文章，通过具体实践帮助读者更快的上手docker。

## 安装docker
> 本文以ubuntu16版本为例。

### 开始安装
由于apt官方库里的docker版本可能比较旧，所以先卸载可能存在的旧版本，再进行更新。
```
 sudo apt-get remove docker docker-engine docker-ce docker.io
 
 sudo apt-get update
```
```
sudo apt-get install -y apt-transport-https ca-certificates curl software-properties-common
```
安装以下包以使apt可以通过HTTPS使用存储库
```
sudo apt-get install -y apt-transport-https ca-certificates curl software-properties-common
```
添加Docker官方的GPG密钥
```
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
```

使用下面的命令来设置stable存储库
```
sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
```

再更新一下apt包索引：
```
sudo apt-get update
```
安装最新版本的Docker CE：
```
sudo apt-get install -y docker-ce
```
在生产系统上，可能会需要应该安装一个特定版本的Docker CE，而不是总是使用最新版本：
```
apt-cache madison docker-ce
sudo apt-get install docker-ce=<VERSION>
```
### 验证docker
查看docker服务是否启动
```
systemctl status docker
```
经典的hello world
```
sudo docker run hello-world
```
运行成功，如下图所示：
![image](http://oum6ifofe.bkt.clouddn.com/image/docker-hello.jpg)

## 应用部署
1. 打包应用
这一步骤主要是拿到前端打包后的文件，具体过程在本文不阐述。读者可以通过scp协议进行上传，也可以通过pm2等自动化工具进行上传。
2. 安装Nginx
如读者的服务器上没有安装nginx，请先安装。
```
docker pull nginx
```
为了使用方便，用docker-compose。

- 创建docker-compose.yml
```
version: '1'
services:
  # 服务名称
  nginx:
    # 镜像:版本
    image: nginx:latest 
    # 映射容器80端口到本地3003端口
    ports:
     - "3003:80"
    # 数据卷 映射本地文件到容器
    volumes:
    # 映射nginx.conf文件到容器的/etc/nginx/conf.d目录并覆盖default.conf文件
     - ./nginx.conf:/etc/nginx/conf.d/default.conf
    # 映射build文件夹到容器的/usr/share/nginx/html文件夹
     - ./dist:/usr/share/nginx/html
    # 覆盖容器启动后默认执行的命令。
    command: /bin/bash -c "nginx -g 'daemon off;'"
```
完成后执行```docker-compose up -d``` 就会自动创建容器并在后台运行了

**常用命令：**

 ```docker-compose down``` 关闭并删除容器
 
```docker-compose ps``` 查看容器状态

```docker-compose exec SERVICE COMMAND``` 可以用来进入容器内部进行一些操作,
比如 ```docker-compose exec nginx bash```

- 静态资源配置

我们先把打包好的dist文件夹上传至服务器,和docker-compose.yml 放在一起
在 docker-compose.yml 文件已经配置了会把同目录的 dist 文件夹映射到nginx容器的/usr/share/nginx/html文件夹， 这个文件夹就是默认的静态资源文件夹。

- 配置Nginx
nginx镜像有一个默认的配置文件default.conf
```
server {
    listen       80;
    server_name  localhost;

    #charset koi8-r;
    #access_log  /var/log/nginx/host.access.log  main;

    location / {
        root   /usr/share/nginx/html;
        index  index.html index.htm;
    }

    #error_page  404              /404.html;

    # redirect server error pages to the static page /50x.html
    #
    error_page   500 502 503 504  /50x.html;
    location = /50x.html {
        root   /usr/share/nginx/html;
    }

    # proxy the PHP scripts to Apache listening on 127.0.0.1:80
    #
    #location ~ \.php$ {
    #    proxy_pass   http://127.0.0.1;
    #}

    # pass the PHP scripts to FastCGI server listening on 127.0.0.1:9000
    #
    #location ~ \.php$ {
    #    root           html;
    #    fastcgi_pass   127.0.0.1:9000;
    #    fastcgi_index  index.php;
    #    fastcgi_param  SCRIPT_FILENAME  /scripts$fastcgi_script_name;
    #    include        fastcgi_params;
    #}

    # deny access to .htaccess files, if Apache's document root
    # concurs with nginx's one
    #
    #location ~ /\.ht {
    #    deny  all;
    #}
```
> 注：默认的配置有一个问题, 在非首页的路由页面刷新就会报404错误。原因在于这些前端静态文件在运行时会更改浏览器uri而又不真的希望服务器对这些uri去作响应，如果此时刷新浏览器，服务器收到浏览器发来的uri就去寻找资源，这个uri在服务器上是没有对应资源，结果服务器因找不到资源就发送403错误标志给浏览器。

所以，我们要做的调整是：浏览器在使用这个应用期间，无论uri更改与否，服务器都发回index.html这个页面就行。

3.创建nginx.conf文件
```
# gzip设置
gzip on;
gzip_vary on;

gzip_comp_level 6;
gzip_buffers 16 8k;

gzip_min_length 1000;
gzip_proxied any;
gzip_disable "msie6";
#gzip_http_version 1.0;
gzip_types text/plain text/css application/json application/x-javascript text/xml application/xml application/xml+rss text/javascript application/javascript;

server {
    listen       80;
    server_name  localhost;

    #charset koi8-r;
    #access_log  /var/log/nginx/host.access.log  main;

    location / {
        root   /usr/share/nginx/html;
        index  index.html index.htm;
        # 其作用是按顺序检查文件是否存在，返回第一个找到的文件或文件夹（结尾加斜线表示为文件夹），如果所有的文件或文件夹都找不到，会进行一个内部重定向到最后一个参数。
        try_files $uri /index.html;
    }

    #error_page  404              /404.html;

    # redirect server error pages to the static page /50x.html
    #
    error_page   500 502 503 504  /50x.html;
    location = /50x.html {
        root   /usr/share/nginx/html;
    }

    # proxy the PHP scripts to Apache listening on 127.0.0.1:80
    #
    #location ~ \.php$ {
    #    proxy_pass   http://127.0.0.1;
    #}

    # pass the PHP scripts to FastCGI server listening on 127.0.0.1:9000
    #
    #location ~ \.php$ {
    #    root           html;
    #    fastcgi_pass   127.0.0.1:9000;
    #    fastcgi_index  index.php;
    #    fastcgi_param  SCRIPT_FILENAME  /scripts$fastcgi_script_name;
    #    include        fastcgi_params;
    #}

    # deny access to .htaccess files, if Apache's document root
    # concurs with nginx's one
    #
    #location ~ /\.ht {
    #    deny  all;
    #}
}
```

将docker-compose.yml里的 
```
# - ./nginx.conf:/etc/nginx/conf.d/default.conf 
```
注释去掉 用nginx.conf覆盖默认的配置。再执行
```
docker-compose down
docker-compose up -d
```
现在我们就能访问3003的应用了。

如果出现docker容器Cannot connect to the Docker daemon. Is the docker daemon running on this host?的问题

原因在于权限问题。需要root用户才能运行

解决方法：
第一种：
直接在命令行中执行sudo命令
```
sudo docker-compose up -d
```
第二种：
Docker 需要用户具有 sudo 权限，为了避免每次命令都输入sudo，可以把用户加入 Docker 用户组：
```
sudo usermod -aG docker $USER
```

Docker 是服务器----客户端架构。命令行运行docker命令的时候，需要本机有 Docker 服务。如果这项服务没有启动，可以用下面的命令启动

```
# service 命令的用法
sudo service docker start

# systemctl 命令的用法
sudo systemctl start docker
```

### 参考文献

- [一个前端的docker笔记](https://github.com/riskers/blog/issues/30)
- [Docker 微服务教程](http://www.ruanyifeng.com/blog/2018/02/docker-wordpress-tutorial.html)
- [Nginx 容器教程](http://www.ruanyifeng.com/blog/2018/02/nginx-docker.html)
- [ubuntu16.04下安装docker](https://blog.csdn.net/bingzhongdehuoyan/article/details/79411479)
