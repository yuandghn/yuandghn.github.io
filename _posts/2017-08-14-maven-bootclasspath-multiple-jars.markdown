---
layout:     post
title:      "Maven - bootclasspath multiple jars"
subtitle:   ""
date:       2017-08-14 19:26:34
author:     "Echo Yuan"
#header-img: "img/in-post/hello-world/hello-world.gif"
tags:
    - Maven
---
在公司的某个项目的pom.xml中看到了如下配置

```
<plugin>
    <artifactId>maven-compiler-plugin</artifactId>
    <version>3.5.1</version>
    <configuration>
        <source>1.7</source>
        <target>1.7</target>
        <encoding>utf8</encoding>
        <compilerArguments>
            <verbose />
            <bootclasspath>
                ${java.home}\lib\rt.jar;${java.home}\lib\jce.jar
            </bootclasspath>
        </compilerArguments>
        <fork>true</fork>
    </configuration>
</plugin>
```

问了几个人，竟然没有人能说清楚为啥要加这项compilerArguments的配置。

好吧，那就手动移除掉，然后`mvn compile`下试试。

不期而遇，build failure了。

看了下输出，原来是项目代码里引用了几个在`rt.jar`中的类
```
com.sun.xml.internal.bind.marshaller.CharacterEscapeHandler
com.sun.xml.internal.messaging.saaj.util.ByteInputStream
```

有的地方甚至只是import了下，并没有真正地使用它们；即使需要用到它们，我相信也有更好的类库可以替代。

先撤消移除再重新`mvn compile -e`下。

```
[ERROR] Failed to execute goal org.apache.maven.plugins:maven-compiler-plugin:3.5.1:compile (default-compile) on project tzxsaas: Compilation failure -> [Help 1]
org.apache.maven.lifecycle.LifecycleExecutionException: Failed to execute goal org.apache.maven.plugins:maven-compiler-plugin:3.5.1:compile (default-compile) on project tzxsaas: Compilation failure
        at org.apache.maven.lifecycle.internal.MojoExecutor.execute(MojoExecutor.java:212)
        at org.apache.maven.lifecycle.internal.MojoExecutor.execute(MojoExecutor.java:153)
        at org.apache.maven.lifecycle.internal.MojoExecutor.execute(MojoExecutor.java:145)
        at org.apache.maven.lifecycle.internal.LifecycleModuleBuilder.buildProject(LifecycleModuleBuilder.java:116)
        at org.apache.maven.lifecycle.internal.LifecycleModuleBuilder.buildProject(LifecycleModuleBuilder.java:80)
        at org.apache.maven.lifecycle.internal.builder.singlethreaded.SingleThreadedBuilder.build(SingleThreadedBuilder.java:51)
        at org.apache.maven.lifecycle.internal.LifecycleStarter.execute(LifecycleStarter.java:128)
        at org.apache.maven.DefaultMaven.doExecute(DefaultMaven.java:307)
        at org.apache.maven.DefaultMaven.doExecute(DefaultMaven.java:193)
        at org.apache.maven.DefaultMaven.execute(DefaultMaven.java:106)
        at org.apache.maven.cli.MavenCli.execute(MavenCli.java:863)
        at org.apache.maven.cli.MavenCli.doMain(MavenCli.java:288)
        at org.apache.maven.cli.MavenCli.main(MavenCli.java:199)
        at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
        at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:57)
        at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
        at java.lang.reflect.Method.invoke(Method.java:606)
        at org.codehaus.plexus.classworlds.launcher.Launcher.launchEnhanced(Launcher.java:289)
        at org.codehaus.plexus.classworlds.launcher.Launcher.launch(Launcher.java:229)
        at org.codehaus.plexus.classworlds.launcher.Launcher.mainWithExitCode(Launcher.java:415)
        at org.codehaus.plexus.classworlds.launcher.Launcher.main(Launcher.java:356)
Caused by: org.apache.maven.plugin.compiler.CompilationFailureException: Compilation failure
        at org.apache.maven.plugin.compiler.AbstractCompilerMojo.execute(AbstractCompilerMojo.java:976)
        at org.apache.maven.plugin.compiler.CompilerMojo.execute(CompilerMojo.java:129)
        at org.apache.maven.plugin.DefaultBuildPluginManager.executeMojo(DefaultBuildPluginManager.java:134)
        at org.apache.maven.lifecycle.internal.MojoExecutor.execute(MojoExecutor.java:207)
        ... 20 more
[ERROR]
```

出错信息不太明显，经过一番搜索后[得知](http://m.blog.csdn.net/u011008029/article/details/72956751)，是bootclasspath中分隔符的问题。在Windows平台上是以`;`来分隔的，而在Linux平台上则应以`:`来分隔
```
<bootclasspath>
    ${java.home}\lib\rt.jar:${java.home}\lib\jce.jar
</bootclasspath>
```

修改后，编译果然通过了。

类比一下，在Windows中设置PATH时各项之间是以`;`来分隔的，而在Linux中是以`:`来分隔的。