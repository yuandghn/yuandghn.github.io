---
layout:     post
title:      "Jenkins - Deploy Maven project to Tomcat"
subtitle:   ""
date:       2017-07-28 20:54:08
author:     "Echo Yuan"
#header-img: "img/in-post/hello-world/hello-world.gif"
tags:
    - Jenkins
    - Maven
    - Tomcat
---
1. 安装[`Deploy to container Plugin`](https://wiki.jenkins.io/display/JENKINS/Deploy+Plugin) 到Jenkins里，虽然plugin的描述里说只支持到Tomcat 7.x，其实Tomcat 8也没问题。

2. 左边菜单`New item` -> 输入item name -> Freestyle project

3. `Source Code Management`

    因为项目的版本控制系统用的是SVN，所以勾选上Subversion。指定好Repository URL后，添加下Credentials，其他选项保持默认。

4. `Build Triggers`

    暂时先选择`Trigger builds remotely`，并定义好`Authentication Token`，这个token的含义描述的很清楚。
    >
    Use the following URL to trigger build remotely: JENKINS_URL/job/demo-project/build?token=TOKEN_NAME or /buildWithParameters?token=TOKEN_NAME
    Optionally append &cause=Cause+Text to provide text that will be included in the recorded build cause.

    最优的方式肯定是在SVN server上写个commit hook script，这样一旦代码有更新就会立即通知Jenkins来运行Job，但是现在一来拿不到SVN server的权限，二来其他人的代码提交习惯很随意、频率较高，频繁地运行Job也没啥好处，所以先采用通过trigger URL或人工在Jenkins里操作的方式来做。

    `Build periodically`也是一个可选的选择，就是cron的表达式要好好理解理解。

5. `Build Environment`

    暂时只勾选上了`Add timestamps to the Console Output`

6. `Build`

    添加个`Invoke top-level Maven target`的build step，`Goals`里输入`clean package`，我们仅仅打个新包。

    好像跳过了`Maven Version`？不，这项设置是很重要的，而且花了好几个小时来搜索和验证。我们先按已知的结果来设置，挖个坑，后面专门写一篇[文章](/2017/07/29/jenkins-cannot-run-program-mvn-error-2-no-such-file-or-directory/)来介绍。

    转到`Manage Jenkins` -> `Global Tool Configuration`里，点击`Maven installations...`，然后添加一个Jenkins所在机器上的本地Maven的引用。这一步其实应该在`New item`之前就做掉的。

    ![add-local-maven](/img/in-post/jenkins-deploy-maven-project-to-tomcat/add-local-maven.png)

    记得要先把`Install automatically`给uncheck掉才会出现`MAVEN_HOME`的输入框哦！

    现在，`Build` -> `Maven version`里就可以选择到`maven-3.3.9`了，不要选`(Default)`。

7. `Post-build Actions`

    添加一个`Deploy war/ear to a container`的action，`WAR/EAR files`里指定成`target/demo-project.war`，`Context path`根据你的需求来定，`Containers`里输入Tomcat里配置的username和password，URL指定到Tomcat的端口那里即可。

    同时，Tomcat的配置文件里也要做些Authentication的设置，否则Jenkins无法将war包部署到remote Tomcat里。

    * 给你使用的username赋予`manager-script`角色的权限

        这个角色的含义是`allows access to the text interface and the status pages`。有时候觉得自己关于Tomcat的知识好匮乏，平常用的都是一些相当皮毛、肤浅的知识。再给自己挖个坑，后面深入地去了解下。

        以下是个例子
        ```
        <-- file: tomcat-dir/conf/tomcat-users.xml -->
        <user username="tomcat" password="tomcat" roles="manager-script"/>
        ```
    * 添加Jenkins所在机器的IP到Tomcat允许的Remote addrs里
        ```
        <-- file: tomcat-dir/webapps/manager/META-INF/context.xml -->
        <Context antiResourceLocking="false" privileged="true" >
          <Valve className="org.apache.catalina.valves.RemoteAddrValve"
                 allow="127\.\d+\.\d+\.\d+|your-jenkins-server's-ip|::1|0:0:0:0:0:0:0:1" />
        ```

     这种方式可行，但不一定是最佳的，毕竟要开放Tomcat的manager权限，这可能会带来一些安全隐患。我想比较理想的方式应该是通过脚本或某种系统将war包分发到各个Tomcat，然后重启生效。

8. `Save` & `Build Now`，看看能否部署成功。










