---
layout:     post
title:      "Maven points Java home to JRE install directory instead of JDK's"
subtitle:   ""
date:       2017-08-15 19:32:59
author:     "Echo Yuan"
#header-img: "img/in-post/hello-world/hello-world.gif"
tags:
    - Maven
---
是在看另外一个问题[maven-bootclasspath-multiple-jars](/2017/08/14/maven-bootclasspath-multiple-jars/)时注意到的，即`${java.home}\lib\rt.jar`中的`java.home`到底指代哪个目录呢？

先`mvn -version`看下效果

```
Apache Maven 3.3.9 (bb52d8502b132ec0a5a3f4c09453c07478323dc5; 2015-11-11T00:41:47+08:00)
Maven home: /Users/echo/Documents/frameworks/apache/apache-maven-3.3.9
Java version: 1.7.0_79, vendor: Oracle Corporation
Java home: /Library/Java/JavaVirtualMachines/jdk1.7.0_79.jdk/Contents/Home/jre
Default locale: en_US, platform encoding: UTF-8
OS name: "mac os x", version: "10.12.3", arch: "x86_64", family: "mac"
```

嗯，`java.home`指到了jre目录下。

再看看`echo $JAVA_HOME`
```
/Library/Java/JavaVirtualMachines/jdk1.7.0_79.jdk/Contents/Home
```

Google了下，stackoverflow上的这个[回答](https://stackoverflow.com/questions/15279586/java-home-in-maven#answer-15279640)说的很好，它里边还引用了这篇[博客](http://javahowto.blogspot.hk/2006/05/javahome-vs-javahome.html)的内容，写的也很棒。

去Oracle的官方文档上看的话，这里[https://docs.oracle.com/javase/tutorial/essential/environment/sysprop.html](https://docs.oracle.com/javase/tutorial/essential/environment/sysprop.html)的描述
> "java.home"	Installation directory for Java Runtime Environment (JRE)

和这里[http://docs.oracle.com/javase/6/docs/api/java/lang/System.html#getProperties()](http://docs.oracle.com/javase/6/docs/api/java/lang/System.html#getProperties())的有些出入
> java.home	Java installation directory

实际在代码里打印`System.getProperty("java.home")`的结果

```
/Library/Java/JavaVirtualMachines/jdk1.7.0_79.jdk/Contents/Home/jre
```

所以，Maven不能背这个锅，哈哈~  人家在[这里](https://maven.apache.org/install.html)也说的很明白
>
Ensure JAVA_HOME environment variable is set and points to your JDK installation

