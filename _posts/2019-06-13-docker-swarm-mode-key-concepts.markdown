---
layout:     post
title:      "Docker Swarm mode key concepts"
subtitle:   ""
date:       2019-06-13 12:05:00
author:     "Echo Yuan"
#header-img: "img/in-post/hello-world/hello-world.gif"
tags:
    - Docker 
    - Swarm
---
Refer to: [https://docs.docker.com/engine/swarm/key-concepts/](https://docs.docker.com/engine/swarm/key-concepts/)
阅读所需时间：4分钟

这篇文章介绍了一些Docker Engine 1.12在集群管理和编排功能方面独有的概念。

#### 什么是Swarm？
内嵌在Docker Engine中的集群管理和编排功能是用[swarmkit](https://github.com/docker/swarmkit/)来构建的。Swarmkit是一个单独的项目，它实现了Docker的编排层并直接用在Docker内部。

一个swarm包含多个Docker hosts，每个host以`swarm mode`运行，分别充当managers(管理成员和代理)和workers(用来运行swarm services)的角色。一个Docker host既可以是个manager也可以是个worker或者同时兼任这两种角色。当你创建一个service的时候，你就定义了它的最佳的状态(副本的数量、网络和存储资源是否对它可用、服务对外暴露的端口等等)。Docker会去维护你所期望的服务状态。例如，当一个worker node变为不可用时，Docker就会调度这个node上的tasks到其它节点上。一个task是一个正在运行着的容器，它是集群服务的一部分并且被swarm manager所管理，它不是一个独立的容器。

Swarm services相对于独立容器的主要优点之一就是你可以修改服务的配置，包括网络和挂载的卷，而无须手动重启服务。Docker会自己来更新这些配置，停掉那些还在使用已过时的配置的task并创建新的task来应用你所期望的配置。

当Docker以`swarm mode`运行时，你仍然可以在参与群集的任何Docker主机上运行独立的容器，当然也可以运行swarm services。在独立容器和swarm services之间的一个关键区别是，仅仅只有swarm managers可以管理这个swarm，而独立的容器则可以被任何Docker守护进程启动。Docker守护进程在集群中可以作为managers、workers或两者兼任。

就像你可以用[Docker Compose](https://docs.docker.com/compose/)去定义和运行容器一样，你也可以用它来定义和运行集群服务[栈](https://docs.docker.com/get-started/part5/)。

我们来继续学习和Docker集群服务相关的概念细节，包括nodes、services、tasks和loan balancing。

#### Nodes
一个node就是参与到swarm里的一个Docker engine的实例。你也可以把它视为一个Docker node。你可以运行一个或多个node在一台单独的物理机或云服务器上，但是生产上部署的集群通常包括分布在多台物理机和云主机上的Docker node。

要部署你的应用到集群中，你需要提交一个服务定义到`manager node`上，manager node会分发工作单元也就是所谓的tasks到worker nodes上。

Manager nodes会负责服务编排和集群管理的工作以维护集群处于所期望的状态。这些Manager nodes会选举出一个单独的leader来实施编排任务。

`Worker nodes`接收和执行manager nodes分配给的任务。默认manager nodes也可以像work nodes一样去运行服务，但是你也可以配置它们只做管理上的工作从而成为一个真正的manager node。每个worker node上都运行有一个agent，它会报告指派给它的tasks。worker node会把分派给它的tasks的状态通知给manager node，以便manager node可以维持每个worker稳定在期望的状态。

#### Service and tasks
`Service`是那些tasks的定义，它可以运行在manager或worker node上。它是集群系统的中心结构和用户与集群交互的根本。

当你要创建一个service的时候，你会指定使用哪个image以及哪些命令会被在运行着的容器中去执行。

在`replicated services`模型下，swarm manager根据你所期望的设置在节点之间分配特定数量的副本任务；而在`replicated service`模型下，swarm manager会在集群中的每一个可用节点上为service运行一个task。

一个task包含一个Docker container和运行在container中的命令，它是swarm的原子调度单元。Manager nodes根据服务规模中设置的副本数来分配tasks给worker nodes，一旦一个task被分配给了某个node，它就不能再被转移到其它的node上，它只能在这个node上运行或者失败。

#### Load balancing
Swarm manager使用`ingress load balancing`来向集群外部暴露服务，Swarm manager可以自动为service分配一个对外可访问的`PublishedPort`，当然你也可以自己指定，只要这个端口没被占用就行。如果你没指定，swarm manager会给服务分配一个30000 ~ 32767之间的端口。

外部组件（如云负载平衡器）可以访问集群中的任何节点的PublishedPort上的服务，无论该节点当前是否正在运行该服务的任务。群集路由中的所有节点都将连接到正在运行的任务实例。

Swarm模式有一个内部的DNS组件可以自动地为swarm中的每个service分配一个DNS入口，swarm manager使用内部的负载均衡来根据service的DNS name在集群内的服务之间分发请求。










