---
layout:     post
title:      "Install Jenkins on CentOS"
subtitle:   ""
date:       2017-07-27 20:48:36
author:     "Echo Yuan"
#header-img: "img/in-post/hello-world/hello-world.gif"
tags:
    - Jenkins
---
话不多说，提高生产力的工具要先行，怎么能忍受项目需要人工来打包和发布这件事。

Jenkins官网在[这](https://jenkins.io)。

1. 首先查看Linux的发行版信息
    ```
    # lsb_release -a
    LSB Version:     :base-4.0-amd64:base-4.0-noarch:core-4.0-amd64:core-4.0-noarch:graphics-4.0-amd64:graphics-4.0-noarch:printing-4.0-amd64:printing-4.0-noarch
    Distributor ID:     CentOS
    Description:     CentOS release 6.5 (Final)
    Release:     6.5
    Codename:     Final
    ```
    其实我也不知道`lsb_release`是啥意思
    ```
    # lsb_release -h
    FSG lsb_release v2.0 prints certain LSB (Linux Standard Base) and Distribution information.
    ```

2. 参照[https://pkg.jenkins.io/redhat-stable/](https://pkg.jenkins.io/redhat-stable/)里的步骤执行。
    ```
    sudo wget -O /etc/yum.repos.d/jenkins.repo https://pkg.jenkins.io/redhat-stable/jenkins.repo
    sudo rpm --import https://pkg.jenkins.io/redhat-stable/jenkins.io.key
    ```
   注意你要install的Jenkins的版本要和你的JDK版本相适应
   * 2.54 (2017-04) and newer: Java 8
   * 1.612 (2015-05) and newer: Java 7

   With that set up, the Jenkins package can be installed with:
    ```
    yum install jenkins
    ```

3. Jenkins的Web UI默认是8080端口，我们改一下吧。修改`/etc/sysconfig/jenkins`文件里的`JENKINS_PORT="8080"`即可，这个跟官方文档里描述的不一样，因为不同的Linux发行版的原因。我是参考的[这里](https://stackoverflow.com/questions/28340877/how-to-change-port-number-for-jenkins-installation-in-ubuntu-12-04#answer-36203490)。

4. Start/Stop
   ```
   sudo service jenkins start/stop/restart
   sudo chkconfig jenkins on
   ```
   但是我这启动失败了
   ```
    Starting Jenkins bash: /usr/bin/java: No such file or directory
                                                               [FAILED]
   ```
   之前在机器上装JDK的人不走寻常路哇，只好自己建个[软链接](https://stackoverflow.com/questions/1951742/how-to-symlink-a-file-in-linux)了。

5. 访问`http://x.x.x.x:18080`，提示要等一会儿，Jenkins正在准备。

6. Unlock Jenkins via Administrator password
    ![Unlock Jenkins](/img/in-post/install-jenkins-on-centos/unlock-jenkins.png)

7. Customize Jenkins，初次使用那就`Install suggested plugins`好了
    ![Customize Jenkins](/img/in-post/install-jenkins-on-centos/customize-jenkins.png)

8. Getting started.
    ![Getting started](/img/in-post/install-jenkins-on-centos/getting-started.png)

   在网络面板中可以看到不停的有`http://x.x.x.x:18080/updateCenter/installStatus?_=1501158675689`之类的请求在发出，挺消耗系统资源的。

   如果install plugins操作没有完成（我是直接关掉了页面，因为安装过程太慢了，可能跟服务器的网络有关），那再次进入`http://x.x.x.x:18080`时会到`Unlock Jenkins`那步。

9. If installation failures
    ![Installation failures](/img/in-post/install-jenkins-on-centos/installation-failures.png)

   第二天来了继续访问`http://x.x.x.x:18080`，其实我是点的`Retry`，结果就直接进去了，没有try。

10. Create first admin user
    ![Create first admin user](/img/in-post/install-jenkins-on-centos/create-first-admin-user.png)

11. Welcome to Jenkins!
    ![Welcome to Jenkins](/img/in-post/install-jenkins-on-centos/welcome-to-jenkins.png)

12. Still install failed plugins before on `http://x.x.x.x:18080/pluginManager/` page.
