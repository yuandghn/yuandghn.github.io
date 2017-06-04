---
layout:     post
title:      "Install open falcon from sourcecode"
subtitle:   ""
date:       2017-06-04 19:46:40
author:     "Echo Yuan"
#header-img: "img/in-post/hello-world/hello-world.gif"
tags:
    - Open-Falcon
---
直接选择的从源码安装，一是本机环境是Mac OS，二是可能会存在一些定制化的需求，二进制版本无法满足，此外对Docker也没有使用经验。在从源码安装Open-Falcon时确实走了点弯路。

一开始是按照[v0.1](https://book.open-falcon.org/zh/index.html)的文档来做的，各个组件都要分编译、部署和启动，不是很方便，当时还想过自己写个脚本来自动化一下，但是考虑到需要迅速对Open-Falcon进行调研就先做罢了。可是等花了一周的时间全部搞定后才发现现在已经出了v0.2版本的[Falcon-Plus](https://github.com/open-falcon/falcon-plus)，![](/img/cry.png)

[Falcon-Plus](https://github.com/open-falcon/falcon-plus)的好处就是对所有的前端组件进行了统一整合，组件的配置项也基本集中在一个配置文件里，编译和部署也无须像v0.1那样自己一个个来了，也变成集中式的了。具体可参考laiwei的简书文章[falcon-plus(v0.2) changelog && 平滑升级方案](http://www.jianshu.com/p/6fb2c2b4d030)。

具体的安装过程就不表了，这里只记录下几个重要的点。

* Go的版本至少要1.8.1
* 请一定设置好GOROOT和GOPATH
* 集中配置项在falcon-plus/config/confggen.sh里
* 针对Mac，如果你的Bash版本<4，请[upgrade to bash 4 in mac osx](http://clubmate.fi/upgrade-to-bash-4-in-mac-os-x/)
* Dashboard仍然需要单独安装，日志文件是var/app.log
* 请把open-falcon-v0.2.0.tar.gz解压出来用，不要在源码下启动
* Falcon+各个组件的日志都在其自己的目录下，如api/logs/api.log、alarm/logs/alarm.log

基于特定的业务需求，我们并不使用Falcon+的所有组件，比如Agent肯定不用。一是因为server的性能监控在我们的业务需求里并不太重要；二是因为门店server都是windows OS，逃…… 甚至后面graph都不要，我们自己来画图。那么，就自己写一个脚本来启动/停止我们需要用到的组件。我都不好意思称之为`脚本`，因为就是一些`./open-falcon start api`的简单罗列，再次逃…… 另外可以写几个类似tail_api_log.sh之类的脚本来方便地查看日志文件。
