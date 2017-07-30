---
layout:     post
title:      "Jenkins cannot run program mvn error 2 - No such file or directory"
subtitle:   ""
date:       2017-07-29 14:18:30
author:     "Echo Yuan"
#header-img: "img/in-post/hello-world/hello-world.gif"
tags:
    - Jenkins
    - Maven
---
在实践前一篇[文章](/2017/07/28/jenkins-deploy-maven-project-to-tomcat/)里的操作步骤时，这个`Build` -> `Invoke top-level Maven target` -> `Maven Version`选项最开始选择的是`(Default)`，因为部署Jenkins的服务器上已经配置好Maven了，不想让Jenkins自己再install一个，所以天真地认为Jenkins应该会正确地找到它的。

可惜事与愿违

```
No changes for https://svn.yourcompany.com:8888/svn/demo-project since the previous build
[deploy-demo-project] $ mvn clean package
FATAL: command execution failed
java.io.IOException: error=2, No such file or directory
	at java.lang.UNIXProcess.forkAndExec(Native Method)
	at java.lang.UNIXProcess.<init>(UNIXProcess.java:247)
	at java.lang.ProcessImpl.start(ProcessImpl.java:134)
	at java.lang.ProcessBuilder.start(ProcessBuilder.java:1029)
Caused: java.io.IOException: Cannot run program "mvn" (in directory "/var/lib/jenkins/workspace/deploy-demo-project"): error=2, No such file or directory
	at java.lang.ProcessBuilder.start(ProcessBuilder.java:1048)
	at hudson.Proc$LocalProc.<init>(Proc.java:245)
	at hudson.Proc$LocalProc.<init>(Proc.java:214)
	at hudson.Launcher$LocalLauncher.launch(Launcher.java:850)
	at hudson.Launcher$ProcStarter.start(Launcher.java:384)
	at hudson.Launcher$ProcStarter.join(Launcher.java:395)
	at hudson.tasks.Maven.perform(Maven.java:367)
	at hudson.tasks.BuildStepMonitor$1.perform(BuildStepMonitor.java:20)
	at hudson.model.AbstractBuild$AbstractBuildExecution.perform(AbstractBuild.java:735)
	at hudson.model.Build$BuildExecution.build(Build.java:206)
	at hudson.model.Build$BuildExecution.doRun(Build.java:163)
	at hudson.model.AbstractBuild$AbstractBuildExecution.run(AbstractBuild.java:490)
	at hudson.model.Run.execute(Run.java:1735)
	at hudson.model.FreeStyleBuild.run(FreeStyleBuild.java:43)
	at hudson.model.ResourceController.execute(ResourceController.java:97)
	at hudson.model.Executor.run(Executor.java:405)
Build step 'Invoke top-level Maven targets' marked build as failure
Finished: FAILURE
```

明显Jenkins时找不到系统里的`mvn`命令。咋办？先Google下呗！精准地进入了Stackoverflow上的这个[回答](https://stackoverflow.com/questions/26906972/cannot-run-program-mvn-error-2-no-such-file-or-directory)。

![a-good-answer-on-stackoverflow](/img/in-post/jenkins-cannot-run-program-mvn-error-2-no-such-file-or-directory/a-good-answer-on-stackoverflow.png)

我觉得人家说的很好，但是还要看自己能不能搞的来。打心眼儿里我是希望能用`locally installed Maven on master/slave`，不想让Jenkins自己再install个Maven，自己配的Maven便于控制settings.xml呀、repository dir呀。

Jenkins安装好后，确实会创建一个名为`jenkins`的user和group到系统里面，`ls -al /var/lib/jenkins`一下就能看出来。

```
drwxr-xr-x  14 jenkins jenkins  4096 Jul 29 15:36 .
drwxr-xr-x. 19 root    root     4096 Jul 27 18:19 ..
drwxr-xr-x   2 jenkins jenkins  4096 Jul 27 20:26 .oracle_jre_usage
-rw-r--r--   1 jenkins jenkins    47 Jul 30 16:54 .owner
-rw-r--r--   1 jenkins jenkins  1626 Jul 29 15:36 config.xml
-rw-r--r--   1 jenkins jenkins   973 Jul 28 10:58 credentials.xml
-rw-r--r--   1 jenkins jenkins   214 Jul 28 13:57 github-plugin-configuration.xml
drwxr-xr-x   2 jenkins jenkins  4096 Jul 27 20:27 userContent
drwxr-xr-x   3 jenkins jenkins  4096 Jul 28 09:10 users
drwxr-xr-x   3 jenkins jenkins  4096 Jul 28 16:18 workspace
```
当然，从`cat /etc/group | grep jenkins`里也能看出来。

既然jenkins是个用户，那我是否可以以此身份来登录CentOS，并亲自在`/var/lib/jenkins/workspace/deploy-demo-project`目录下执行mvn命令来看看效果呢？

还真尝试了下，可惜不行。因为它仅仅是一个`service account`，没有password，也没有shell。我是从[这里](https://stackoverflow.com/questions/18068358/cant-su-to-user-jenkins-after-installing-jenkins)知道的。

那会不会是因为jenkins用户没有执行mvn命令的权限呢？也不是。

```
-rwxr-xr-x 1 root root 7383 Jul 10 15:32 mvn
```

唔~ 有点儿陷入僵局了。

眼见天色已晚，那就先让Jenkins自己install个Maven吧，哈哈~ 去`Manage Jenkins` -> `Global Tool Configuration` -> `Maven installations...`里操作下即可。

![install-maven-by-jenkins](/img/in-post/jenkins-cannot-run-program-mvn-error-2-no-such-file-or-directory/install-maven-by-jenkins.png)

再回到我们的Job配置里，现在`Maven Version`的列表中多了一个`maven-3.5.0`，选择它，然后Build下Job，呵呵，成功了！

再然后，在和一位同事的交流中得知，还可以用前一篇[文章](/2017/07/28/jenkins-deploy-maven-project-to-tomcat/)中描述的那种方式来做，这样就无须Jenkins install新的Maven了。试了下，确实可以，但是Job的`Console output`里会有些别样的内容出来。

```
14:21:05 No changes for https://svn.yourcompany.com:8888/svn/demo-project since the previous build
14:21:05 [deploy-demo-project] $ /data/apache-maven-3.3.9/bin/mvn clean package
14:21:05 which: no javac in (/data/apache-maven-3.3.9/bin:/sbin:/usr/sbin:/bin:/usr/bin)
14:21:05 Warning: JAVA_HOME environment variable is not set.
14:21:06 [INFO] Scanning for projects...
```

`javac`确实不在这几个bin下面，但`JAVA_HOME`在当前登录用户(root)下是存在的。奇怪的是Jenkins为啥只去这几个bin目录`/data/apache-maven-3.3.9/bin:/sbin:/usr/sbin:/bin:/usr/bin`下面找可执行文件呢？

于是，有个想法冒出来了。如果我在`/usr/bin`下建个`mvn`的软链接指到`/data/apache-maven-3.3.9/bin/mvn`会怎么样呢？Just do IT！

嘎嘎~ 竟然可行！但是以我这贫乏的Linux知识是难以解释通了，

![escape](/img/cry.png)

如果您能看透这一切，请留言让我知晓。Thanks in advance!