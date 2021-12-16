---
layout:     post
title:      "What the hell Pod's containerPort is used for?"
subtitle:   ""
date:       2021-12-16 20:46:08
author:     "Echo Yuan"
tags:
    - Kubernetes
    - Pod
    - containerPort
---
为了准备CKA(Certified Kubernetes Administrator)考试，最近在练习kubernetes的基础操作。

大概一年多前，曾经学过Docker和一些k8s的知识，也在实际项目里写了不少的k8s资源配置文件，感觉好像差不多都懂的。可最近学到Pod这一节时，对`containerPort`心中又起了疑问，是那种似懂非懂的感觉。问了下身边的同事，感觉解释的也不是十分满意。Google了一下，再配上实际操作的例子，终于把它搞懂了！

那`containerPort`究竟是干啥用呢？答曰：没啥实际用途，纯粹就是为了记录一下，方便后来人了解某个应用自身都暴露了/监听着哪些端口。

先瞅一下[官方文档](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.21/#container-v1-core)里的描述 

    List of ports to expose from the container. Exposing a port here gives the system additional information about the network connections a container uses, but is primarily informational. Not specifying a port here DOES NOT prevent that port from being exposed. Any port which is listening on the default "0.0.0.0" address inside a container will be accessible from the network. Cannot be updated.

这句`but is primarily informational`很重要，表明它就是单纯地show给别人看的；

这句`Not specifying a port here DOES NOT prevent that port from being exposed`更重要，是说就算你没指定`containerPort`，也不可能阻止应用自身的端口暴露出去，更进一步的意思就是，就算你指定了一个和应用自身暴露的端口不一样的端口，也没啥鸟用，Pod不会因此而做一个端口转发的。

所以

* container里的应用监听了几个端口，这个container就会对外暴露几个端口

  比如Spring Boot应用里的tomcat启动后默认监听8080端口，Spring actuator可以指定另外一个18888端口，那这个container就会对外暴露8080和18888这两个端口，这是改变不了的事实。

* Pod里可以包含多个containers，每个container暴露的端口同时也是Pod对外暴露的端口，Pod只是一个逻辑上的概念。

  但要注意，这些containers里的所有应用监听的端口一定不能重复，不然会有container启动失败。

* 虽然`containerPort`没啥实际用处，但还是强烈建议加上，因为它可以告诉后面的人你的应用究竟监听了哪些端口，你总不能要求人家去看你的应用的代码吧

举一个实际的例子
```
apiVersion: v1
kind: Pod
metadata:
  name: task-pv-pod
spec:
  containers:
  - name: task-pv-container-1
    image: nginx
    ports:
    - containerPort: 1880
      name: "http-server"
  - name: task-pv-container-2
    image: nginx
    ports:
    - containerPort: 2880
      name: "http-server-2"
```      

`1880`和`2880`不会起到任何作用，这俩nginx监听的都是80端口，自然地，`task-pv-container-2`会启动失败。

参考文档：

[https://stackoverflow.com/questions/57197095/why-do-we-need-a-port-containerport-in-a-kuberntes-deployment-container-definiti](https://stackoverflow.com/questions/57197095/why-do-we-need-a-port-containerport-in-a-kuberntes-deployment-container-definiti)

[https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.21/#container-v1-core](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.21/#container-v1-core)
