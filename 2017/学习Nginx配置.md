---
title: 学习Nginx相关配置
category: BackEnd
tags: [nginx,proxy,Node.js]
---

### Nginx的作用
- 反向代理，实现简单的负载均衡： 如果有多台服务器或者一台服务器多个端口，可以考虑用nginx。
- 静态资源缓存：把一些静态资源（如静态页面，js等资源文件）放到nginx里，可以极大的提高服务的性能。

>Nginx是一款轻量的Web网页开发服务器，反向代理服务器，以稳定性，丰富的功能级，低系统资源消耗闻名。Nginx有着出色的HTTP反向代理能力，应该放在各种Web开发语言的前面。

### Nginx和单进程NodeJS比较
1. 静态文件性能
NodeJS处理静态文件，受限于单线程异步IO模型。在多核处理器上，应该尽量使用Nginx来处理静态文件。
2. 反向代理规则
反向代理服务多样，让配置好的机器分担更多的压力。
3. 稳定性和转发性能
Nginx在同样的负载，相比NodeJS占用更好的CPU和内存资源
4. 安全性
减少DOS攻击
5. 运维管理
监听不同的端口，通过反向代理配置，将多个站点域名指向一个机器。

### Nginx在服务器上的配置
1. 删除appache,首先检测是否有apache
```sh
sudo service apache2 stop
sudo service apache stop
```

2. 删除apache
```sh
update-rc.d -f apache2 remove 
```
3. 移除apache
```sh
sudo agt-get remove apache2
```
4. 更新列表
```sh
sudo apt-get update
```
5. 安装nginx
```sh
sudo apt-get install nginx
```
6. 进入nginx目录
```sh
cd /etc/nginx

cd conf.d
```
7. 命名
```sh
sudo vi perkinzone-com-8081.conf
```
8. 配置
```sh
upstream perkinzone {
        server 127.0.0.1:8081
}

server {
        listen:80;
        server_name:localhost;

        location / {
                proxy_set_header X-Real-IP $remote_addr;
                proxy_set_header X-Forward-For $proxy_add_xforwarded_for;
                proxy_set_header Host $http_host;
                proxy_set_header X-Nginx-Proxy true;

                proxy_pass http://perkinzone;  //配置指令来实现代理，把域名配置到代理的名字上面，或者是应用的名字上面
                proxy_redirect off;
        }
}
```
9. 然后再配置nginx.conf
```sh
sudo vi nginx.conf
```
10. 查看是否被注释，注释则打开
```sh
include /etc/nginx/conf.d/*.conf;
include /etc/nginx/sites-enabled/*;
```
11. 检测配置文件是否有错误
```sh
sudo nginx -t
```
12. 重启nginx
```sh
sudo nginx -s reload

sudo service nginx restart

```
13. 让nginx对外透明，编辑nginx.conf
```sh
sudo vi nginx.conf   //打开 server_tokens off;
14. 再设置防火墙
