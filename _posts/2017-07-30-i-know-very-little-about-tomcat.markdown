---
layout:     post
title:      "I know very little about Tomcat"
subtitle:   ""
date:       2017-07-30 16:33:55
author:     "Echo Yuan"
#header-img: "img/in-post/hello-world/hello-world.gif"
tags:
    - Tomcat
---
1. 如何进入管理界面？

    在tomcat-dir/conf/tomcat-users.xml文件中为你要使用的登录账户赋予`manager-gui`角色的权限。
    ```
    <user username="tomcat" password="tomcat" roles="manager-gui"/>
    ```
2. What's the difference between catalina.sh and startup.sh/shutdown.sh under bin directory?

    startup.sh/shutdown.sh只是简单包了一下catalina.sh，实际还是调用的catalina.sh start/stop

3. How to force the Tomcat to shutdown?

    经常碰到shutdown.sh不能真正地停掉Tomcat的情况，一般都是因为Tomcat无法正常关闭一些可能造成内存泄露的资源所导致的，像我这里有遇到过MQ相关的、Netty Thread pool相关的、Local thread相关的等等。说到底，还是因为我们自己没能正确地、及时地释放掉这些资源，毕竟一个初始状态的Tomcat，shutdown起来还是妥妥滴。

    但事已至此，该停还得停，目前的方式估计只有kill进程了。自己手动做有点儿麻烦，Tomcat自身其实提供了这个功能。

    在`bin/setenv.sh`文件(如果没有这个文件请创建)中添加一行

    ```
    CATALINA_PID="/tmp/catalina.pid"
    ```
    用于存储Tomcat的PID，路径和文件名可以根据自己的需求来定。以后想强制Tomcat shutdown就可以执行以下命令了。
    ```
    sh ./bin/catalina.sh stop 15 -force
    ```
    我这的输出如下
    ```
    ......
    The stop command failed. Attempting to signal the process to stop through OS signal.
    Tomcat did not stop in time.
    To aid diagnostics a thread dump has been written to standard out.
    Killing Tomcat with the PID: 21958
    The Tomcat process has been killed.
    ```

    `stop`指令后面的参数的单位是秒，作用是给予Tomcat一定的时间来做清理工作，这个时间一到才会被强制shutdown。`-force`的作用不说也明了。

    关于`CATALINA_PID`的描述
    > Path of the file which should contains the pid of the catalina startup java process, when start (fork) is used



未完待续...



