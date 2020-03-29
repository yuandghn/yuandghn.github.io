---
layout:     post
title:      "Some problems in learning Docker Swarm"
subtitle:   ""
date:       2020-03-29 23:00:00
author:     "Echo Yuan"
tags:
    - Docker
    - Swarm
---

去年学过一段Docker Swarm，但没机会用到实际的项目中，就先停在那了。今年打算重新拾起来，所以来温习一下。过程却比去年挫折些，遇到了一些去年没见过的问题，哈哈~

去年是跟着v18.09的[文档](https://docs.docker.com/v18.09/get-started/)学习的，担心最新版本的会跟它有差别，而且考虑到staging、production上的Docker版本一般都不会怎么新，所以就继续18.09了。

以下是碰到的几个问题。

#### Swarm里的service启动之后啥输出都没有，一会儿就变成Shutdown的了

原因是我的`docker-compose.yml`里没有配`command: java -jar /app/usercenter-service.jar`！！！因为`Dockfile`里没有这个命令了，为了能做到`JAVA_OPTS`在`.env`里进行配置，所以出现了这么诡异的事情。

我是怎么发现的呢？是在`docker ps`时发现部署的服务在`COMMAND`那一列的值是`bash`，而不是像其他那样的`java -jar…`，也算是巧合吧，不然估计还要查好久。

#### `docker service logs -t -f SERVICE_NAME`看不到应用实时输出的日志

这是因为没在`logback-spring.xml`里给正在测试用的`spring profile`加上`CONSOLE`这个`appender`，所以应用的日志无处输出。而我们在其它环境`tail logs/stdout.log`时没问题是因为它们是`ROLLING_FILE`，且已经将`container`里的应用日志文件映射到了宿主机上。

真是哭笑不得呀，浪费了一天的时间在这个问题上。

另外要考虑的问题是，`ROLLING_FILE`感觉没必要了，因为无法映射到主机上了，container一销毁里面rolling的那些files也就跟着没了。有同事说可以映射到宿主机的`/var/log`下，不知道是不是`best practice`。

#### `docker stack deploy`后ps一下stack发现service报了`invalid mount config for type`

原因是在Swarm里存在多个nodes的场景下已经不能在`docker-compose.yml`里使用`volumns`来挂载配置文件和日志目录映射了。#2里解决了日志目录映射的问题，而挂载配置文件可以换用[docker configs](https://docs.docker.com/engine/swarm/configs/)来做。

#### 创建VirtualBox vm时会校验当前的`boot2docker`的版本，如果out of date的话就会去下载最新版本的`boot2docker`，但是下载速度太慢了

可以用专门的下载工具先下载下来，然后拷贝到`/Users/echo/.docker/machine/cache`下，并在create vm时指定好它

`docker-machine create YOUR-MACHINE-NAME --virtualbox-boot2docker-url /Users/echo/.docker/machine/cache/boot2docker.iso`

参考自[https://github.com/docker/machine/issues/3346](https://github.com/docker/machine/issues/3346)

#### boot2docker.iso 的 v18.09.0 版本存在问题

curl user-service在VirtualBox vm中不起作用，且在宿主机的浏览器里也无法访问user-service对外暴露的端口和服务，但是在内网的测试服务器和我本机组成的swarm中却不存在这个问题。

以下两个帖子说的就是v18.09.0 存在的问题

[https://forums.docker.com/t/get-started-part-4-connection-refused-from-node-on-virtual-machine/62511/9]()https://forums.docker.com/t/get-started-part-4-connection-refused-from-node-on-virtual-machine/62511/9)

[https://github.com/docker/machine/issues/4608](https://github.com/docker/machine/issues/4608) 

换用v18.09.6就没问题了。

其实去年学习Docker Swarm的时候我并没有碰到这个问题，原因是当时用的boot2docker是v19.03.5，而这几天重新学习的时候因为#4的问题而特意指定了和本机Mac Docker Desktop版本一致的18.09.0，当时是担心boot2docker版本比本机Docker版本高太多的话可能会有一些潜在的问题，结果反而弄巧成拙，花了一天多的时间在尝试解决这个问题上。

#### scp本地文件到VirtualBox vm中

```bash
# Copy file to node's home dir (only required if you use ssh to connect to manager and deploy the app)
docker-machine scp docker-compose.yml myvm1:~
``` 

#### 手动创建外部的公共network

```bash
docker network create micro-service -d overlay --scope swarm
```    

#### `docker stack ps STACK` 看不到对外暴露的端口，特别是没指定host port而让docker自己分配的时候，需要用`docker stack services STACK`来看对外暴露的端口

#### 可以用service name来互相访问吗？
    
可以的，在`docker-compose.yml`中把host port去掉，只保留container port，让docker自己来分配对外暴露的端口，然后同一个network中的services就可以互相用`service-name:container-port`来访问了。什么？也不想加`container-port`？那你的service以`80`端口来启动好了，当然这只是视觉上看不到而已。

#### Virtual Box - auto capture keyboard

嫌在宿主机上`docker-machine ssh myvm1`麻烦，想直接通过VirtualBox进到vm里面，结果选了`auto capture keyboard`，键盘和鼠标只能在vm里使用了，差点退不出来，还好搜索了下知道怎么解决了。

其实人家的弹框里也有提示的，只是没仔细阅读，不然也不至于这么狼狈。

![好](/img/in-post/some-problems-in-learning-docker-swarm/virtual-vm-auto-capture-keyboard.png)

按 Left Command 键就行了








