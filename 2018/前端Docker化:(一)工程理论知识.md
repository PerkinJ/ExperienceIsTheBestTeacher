# 前端Docker化:(一)工程理论知识
> 最近本人在学docker，虽然网上关于docker的文档或者资料很多，但针对于前端开发人员的或者前端实践的文章并不多，故总结了这一系列文章，通过具体实践帮助读者更快的上手docker。

## docker的理念与场景
Docker 属于 Linux 容器的一种封装，提供简单易用的容器使用接口。它是目前最流行的 Linux 容器解决方案。

Docker 将应用程序与该程序的依赖，打包在一个文件里面。运行这个文件，就会生成一个虚拟容器。程序在这个虚拟容器里运行，就好像在真实的物理机上运行一样。有了 Docker，就不用担心环境问题。
### docker 的理念
#### 1. build
```
FROM centos:centos6
MAINTAINER yourname@example.com

RUN yum -y update; yum clean all
RUN yum -y install epel-release; yum clean all
RUN yum -y install nodejs npm; yum clean all

# copy 程序代码到容器的/src 下
ADD . /src

RUN cd /src; npm install

EXPOSE 8080

CMD ["node", "/src/index.js"]
```
上面就是一个简单的 Node.js 应用的 build 描述文件，docker 根据这个描述文件来 build 出来一个镜像，一个可以到处运行的镜像。

FROM , 它的意思是要从哪里作为基础镜像来制作你现有的镜像。

**docker 具有的这种能够打包环境能力，是它能到处运行的关键。**

#### 2. ship

与github类似，docker拥有dockerhub。把制作好的镜像 docker push 到 dockerhub或者私有 registry（你可以理解为 git 对应的 GitLab ），用户通过 docker pull 到本地。这也是 docker 进行版本控制的一种方式

差别：

传统的发布只是把代码或者程序发布到线上，如果不涉及类似 Node.js 版本升级，npm 模块升级，环境变量更迭及其它代码之外的的变更，其实是凸显不出 docker 的优势的。

docker 不仅分发镜像，也可以分发镜像的描述文件 DockerFile , 比如你想让测试人员在你的环境里面去测试和复现 bug，不用提供个头稍微大一些的镜像，给他一个文本文件就好了。

  
#### 3. run
> 对于传统前端来讲，把静态文件放到 web 服务器的对应目录就可以了，不涉及其它动作。对于 Node.js 类应用来讲可能会复杂一点，不仅仅是把应用代码发布上去就能生效，可能涉及到应用关停等影响服务可用性的动作。

docker优势：
- 可以运行在任何可以运行 docker 容器的机器上。如果你的应用就是固定的两台设备，这体现不出 docker 的优势，但如果你的应用分布在不同的地方，甚至不同的云服务上，都可以保证正常的秒级启动。那这也给我们在突发流量的时候可以迅速实现扩容。
- 非常轻量，和传统的进程没太大差别，这也是相对于传统的 vm 非常大的一个优势
- 资源隔离， 保证各个应用之间不相互影响，同时充分利用机器资源

### Docker的应用场景
1. 需要重复配置统一的环境
    - 一个团队开发同一个程序，需要一致的开发环境，在之前你可能每个成员在自己电脑上配置一遍，但使用 docker 基本是配置一遍，其他同学共享就可以了
    - 减少持续集成和测试的难度
    - 减少开发，测试，线上因为环境不一致而出现的问题的几率
    - 世界上有很多知名和成熟的镜像，我们可以拿来用就可以了，提高效率
2. 快速，高效利用当前资源
    - 有时候一个容器就能完成的事情，我们申请一台 vm，资源有些浪费
    - 生产一台 vm 是分钟级别的，而一台容器却是秒级的，这种优势在大规模需要迅速实现扩容才会凸显出来。
3. 需要资源隔离的场景
    - 比如一些多租户的环境里面，一个用户的失控比如程序把内存吃光，把带宽打满都是常有的事情，而 docker 就可以很好的在这些方面做好控制
4. 需要快速动态的伸缩容量的场景
    - 比如说大促，开始之前迅速扩容，结束之后迅速销毁
5. 微服务

## Docker架构
### virtualbox的作用
docker 提出的容器技术并不是 docker 首创，在之前就有了 LXC 等基于 Linux 内核特性（ cgroups 和 namespace ）的容器技术，而 docker 可以看做是加强版的 LXC ，增加了类似更强的可移植性，版本控制，类库可分享（ docker hub ） 等一些列简化容器操作功能。

docker 因为底层依赖的原因目前只能跑在具有相关特性的 Linux 上面，而要在不支持这些特性的操作系统上安装 docker，就需要额外多装一个 Linux 的环境了。

![image](https://img.alicdn.com/tps/TB1xd6kLpXXXXboXXXXXXXXXXXX-672-427.png)

### docker-machine的作用
docker-machine 的出现就是为了解决 docker 可以安装在更多的地方的一个管理工具。你也可以理解是大大的扩展了 boot2docker ，为 docker 的方便安装，官方开发的一个工具。它帮你屏蔽了一些安装的麻烦，让你更好更快的用上 docker。

## docker run的基本流程
- 使用 docker-machine 创建一个 vm
```
docker-machine create -d virtualbox default

```
- 启动 vm
```
docker-machine start default
```
- 设置环境变量
在 Linux 上是不需要的，但是通过 docker-machine 创建的则需要
```
eval "$(docker-machine env default)"
```
docker 是 C/S 架构，docker client 把命令发给 docker daemon ，然后 daemon 进行具体的操作。在执行 docker run 的过程中，docker 先会检查本地是否有相关镜像，如果没有就会去 docker hub 进行检索并下载到本地。
![image](https://img.alicdn.com/tps/TB1UifkLpXXXXXAXXXXXXXXXXXX-756-395.svg)

> Kitematic是一个 Docker GUI 工具，它可以在 Mac 上更快速、更简单的运行Docker。

## 参考文献

[Docker 入门教程](http://www.ruanyifeng.com/blog/2018/02/docker-tutorial.html)

[docker 的理念与场景](http://taobaofed.org/blog/2016/01/19/feders-should-kown-some-docker-1/)

[docker 架构](http://taobaofed.org/blog/2016/01/21/feders-should-kown-some-docker-2/)
