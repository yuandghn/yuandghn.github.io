---
layout:     post
title:      "Maven - package for multiple environments"
subtitle:   ""
date:       2017-08-03 18:19:15
author:     "Echo Yuan"
#header-img: "img/in-post/hello-world/hello-world.gif"
tags:
    - Maven
---
在项目打包的时候，我们经常会遇到需要在不同的环境里使用与其相配套的配置文件的需求，不同环境需要的配置文件的数量可能不一样，某些配置文件里的配置项的数量和值也可能不一样。所以，我认为比较简单粗暴的方式是打包时根据目标环境的不同将其所使用的所有配置文件直接覆盖默认的配置文件。

先上一张标准的Maven项目结构图，来自于[introduction-to-the-standard-directory-layout](https://maven.apache.org/guides/introduction/introduction-to-the-standard-directory-layout.html)

![maven-standard-directory-layout](/img/in-post/maven-package-for-multiple-environments/maven-standard-directory-layout.png)

我们把各个环境的配置文件放到`src/main/filters`下，虽然名字上不是特别契合，但是我觉得从某种角度上看也是可以的。也或者`src/main/configs`更合适些。
![maven-standard-directory-layout](/img/in-post/maven-package-for-multiple-environments/filter-by-envs.png)

各个env所使用的配置文件在名字和数量上可能会不一样，就像上图那样。

鉴于我刚才纠结了下`filters`这个命名，我们先来看看Maven中的`filtering`是什么意思。

`filtering`是Maven的[maven-resources-plugin](https://maven.apache.org/plugins/maven-resources-plugin/index.html)中的概念，它的[定义](https://maven.apache.org/plugins/maven-resources-plugin/examples/filter.html)如下

>
Variables can be included in your resources. These variables, denoted by the ${...} delimiters, can come from the system properties, your project properties, from your filter resources and from the command line.

然后，当你在\<resource\>标记中启用`filtering`时
```
...
<resource>
    <directory>src/main/resources</directory>
    <filtering>true</filtering>
</resource>
...
```
输出的所有resources文件中的`${...}`都会被替换成相应的变量的值，它的作用跟Spring中的`PropertyPlaceholderConfigurer`基本上一样，只是时机上不同，可以考虑下哪种方式更适合你。两者之间可能也有些细微的差别，目前还未Get到，:)。

关于resources文件的输出要先阐明
>
The default resource directory for all Maven projects is src/main/resources which will end up in target/classes and in WEB-INF/classes in the WAR. The directory structure will be preserved in the process.

我们继续回到多环境的配置上。

在pom.xml中先添加各个环境对应的profile
```
<profiles>
    <profile>
        <id>dev</id>
        <properties>
            <!-- target-env可以改成其他的名字 -->
            <target-env>dev</target-env>
        </properties>
        <activation>
            <activeByDefault>true</activeByDefault>
        </activation>
    </profile>
    <profile>
        <id>qa</id>
        <properties>
            <target-env>qa</target-env>
        </properties>
    </profile>
    <profile>
        <id>preview</id>
        <properties>
            <target-env>preview</target-env>
        </properties>
    </profile>
    <profile>
        <id>prod</id>
        <properties>
            <target-env>prod</target-env>
        </properties>
    </profile>
</profiles>
```

在Maven命令中使用`-P`参数即可指定要使用的profile的id，如

```
mvn clean package -P qa
```
如果不指定，默认使用`activeByDefault=true`所标识的profile。

在maven-war-plugin中做些修改，官方文档在[adding-filtering-webresources](http://maven.apache.org/plugins/maven-war-plugin/examples/adding-filtering-webresources.html)
```
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-war-plugin</artifactId>
    <version>2.4</version>
    <configuration>
        <archive>
            <addMavenDescriptor>false</addMavenDescriptor>
        </archive>
        <!--
        <filters>
            <filter>src/main/filters/${target-env}/db.properties</filter>
        </filters>
        -->
        <webResources>
            <resource>
                <directory>src/main/filters/${target-env}</directory>
                <targetPath>WEB-INF/classes</targetPath>
                <!-- 要准确地理解filtering的含义 -->
                <!--<filtering>true</filtering>-->
            </resource>
        </webResources>
    </configuration>
</plugin>
```

在这里我们动态地添加了额外的resources，并且它在输出后会覆盖原有的`src/main/resources`目录下的同名文件。这样，当我们指定好`-P`参数后就可以为不同的环境应用与之相配套的配置文件了。

注意到我把filter的相关配置注释掉了吗？因为在这里filter不太好用，它需要明确地指出要使用哪些property file且不能使用通配符来模糊匹配，既然property files也会被拷贝到输出目录，那还是使用Spring的`PropertyPlaceholderConfigurer`来做这件事吧。

按我们现在的配置，你必须指定一个profile作为`activeByDefault`，不然会在copy web resources的时候报错，因为会找不到这个目录`<directory>src/main/filters/${target-env}</directory>`。

话说，某天突发一想法，即pom.xml中的pom指代的是什么意思呢？[官网](https://maven.apache.org/guides/introduction/introduction-to-the-pom.html)上就有介绍
> What is a POM?

> A `Project Object Model` or POM is the fundamental unit of work in Maven.
