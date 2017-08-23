---
layout:     post
title:      "Learning Zipkin"
subtitle:   ""
date:       2017-08-23 19:42:22
author:     "Echo Yuan"
#header-img: "img/in-post/hello-world/hello-world.gif"
tags:
    - Zipkin
---
### Zipkin是啥？

[Zipkin](http://zipkin.io/) is a distributed tracing system. It helps gather timing data needed to troubleshoot latency problems in microservice architectures. It manages both the collection and lookup of this data. Zipkin’s design is based on the [Google Dapper](Google Dapper) paper.

### Quickstart

一开始尝试的是从源码编译运行，过程中下载的依赖很多，不仅限于Java，还有Node.JS；而且可气的是系统里已经有Maven了它也不用，自己额外下载个Maven并把一大堆依赖下载到非`~/.m2/repository`目录里，不爽，也正好因为网络原因卡住了，遂停止改用`self-contained executable jar`。

```
wget -O zipkin.jar 'https://search.maven.org/remote_content?g=io.zipkin.java&a=zipkin-server&v=LATEST&c=exec'
java -jar zipkin.jar
```

Make sure you have had Java8 installed as current JDK version, or you will encountered

```
Exception in thread "main" java.lang.UnsupportedClassVersionError: zipkin/server/ZipkinServer : Unsupported major.minor version 52.0
```

### 启动界面蛮漂亮的

![zipkin-started](/img/in-post/learning-zipkin/zipkin-started.png)

注意到的几个点
* :: Powered by Spring Boot ::         (v1.5.6.RELEASE)
* Starting Servlet Engine: Apache Tomcat/8.5.16
* Initializing Spring embedded WebApplicationContext
* Registering beans for JMX exposure on startup

将一个项目运行所需要的全部依赖打进一个完整的部署包里或许就是将来的趋势，再把运行环境也打打包，不就成Docker做的事情了嘛~  没什么不好，除了比以前要重好多之外。

这只是服务端哦，用于接收、存储和聚合处理数据并提供查询接口和一个Web UI。

Zipkin提供了各个语言的用于收集数据的官方的或社区的[instrumented libraries](http://zipkin.io/pages/existing_instrumentations.html)，Java语言的话，我们用[Brave](https://github.com/openzipkin/brave)。

### Running brave-webmvc-example project

```
git clone https://github.com/openzipkin/brave-webmvc-example.git
```

里面有两个Maven工程分别对应Servlet2.5和Servlet3，我们将Servlet3导入Intellij IDEA并运行一下就可以看到效果。

这个例子很简单，但是全部都是用注解来做的，相对来说还是基于XML的Configuration看着亲切些，因为我们能更轻易地了解和掌控全局配置。

Just do IT.

### Running our webmvc with Brave

在[https://github.com/openzipkin/brave](https://github.com/openzipkin/brave)的README里描述了`Spring XML Configuration`里的做法。

> If you are trying to trace legacy applications, you may be interested in [Spring XML Configuration](https://github.com/openzipkin/brave/blob/master/brave-spring-beans). This allows you to setup tracing without any custom code.

不过里面的url现在404了，正确的应该是[https://github.com/openzipkin/brave/tree/master/spring-beans](https://github.com/openzipkin/brave/tree/master/spring-beans)，点进去，然后打开[pom.xml](https://github.com/openzipkin/brave/blob/master/spring-beans/pom.xml)看看它的`artifactId`，于是就可以在我们自己的项目的pom.xml里添加以下依赖

```
<dependency>
    <groupId>javax.servlet</groupId>
    <artifactId>javax.servlet-api</artifactId>
    <version>3.1.0</version>
    <scope>provided</scope>
</dependency>

<dependency>
    <groupId>io.zipkin.brave</groupId>
    <artifactId>brave-spring-beans</artifactId>
    <version>${brave.version}</version>
</dependency>
```

按照文档里的说明，我们定义一下各个bean

```
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:mvc="http://www.springframework.org/schema/mvc"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
        http://www.springframework.org/schema/beans/spring-beans.xsd
        http://www.springframework.org/schema/mvc
        http://www.springframework.org/schema/mvc/spring-mvc.xsd">

    <bean id="sender" class="zipkin.reporter.okhttp3.OkHttpSender" factory-method="create">
        <constructor-arg index="0" value="http://127.0.0.1:9411/api/v1/spans" />
    </bean>

    <bean id="tracing" class="brave.spring.beans.TracingFactoryBean">
        <property name="localServiceName" value="say-hello-webmvc"/>
        <property name="reporter">
            <bean class="brave.spring.beans.AsyncReporterFactoryBean">
                <property name="sender" ref="sender"/>
                <!-- wait up to half a second for any in-flight spans on close -->
                <property name="closeTimeout" value="500"/>
            </bean>
        </property>
        <property name="currentTraceContext">
            <bean class="brave.context.log4j2.ThreadContextCurrentTraceContext" factory-method="create"/>
        </property>
    </bean>

    <bean id="httpTracing" class="brave.spring.beans.HttpTracingFactoryBean" init-method="getObject">
        <property name="tracing" ref="tracing"/>
    </bean>

</beans>
```

然后会发现有类找不到，因为必须添加以下dependencies才行

```
<dependency>
    <groupId>io.zipkin.brave</groupId>
    <artifactId>brave</artifactId>
    <version>${brave.version}</version>
</dependency>
<dependency>
    <groupId>io.zipkin.reporter</groupId>
    <artifactId>zipkin-sender-okhttp3</artifactId>
    <version>${zipkin-reporter.version}</version>
</dependency>
<dependency>
    <groupId>io.zipkin.brave</groupId>
    <artifactId>brave-context-log4j2</artifactId>
    <version>${brave.version}</version>
</dependency>
```

我们的目的只是想看看Controller中负责处理每个Request的Method前后所花费的时间，所以先去瞅瞅[brave-webmvc-example-Servlet 3](https://github.com/openzipkin/brave-webmvc-example/blob/master/servlet3/src/main/java/brave/webmvc/TracingConfiguration.java)里的配置。

嗯~ 我们需要添加一个`TracingHandlerInterceptor`。

```
<mvc:interceptors>
    <bean class="brave.spring.webmvc.TracingHandlerInterceptor"/>
</mvc:interceptors>
```

这个类在另外一个dependency里

```
<dependency>
    <groupId>io.zipkin.brave</groupId>
    <artifactId>brave-instrumentation-spring-webmvc</artifactId>
    <version>${brave.version}</version>
</dependency>
```

跑一下项目，发现还少两个dependencies

```
<dependency>
    <groupId>org.apache.logging.log4j</groupId>
    <artifactId>log4j-core</artifactId>
    <version>${log4j.version}</version>
</dependency>
<dependency>
    <groupId>org.apache.logging.log4j</groupId>
    <artifactId>log4j-slf4j-impl</artifactId>
    <version>${log4j.version}</version>
</dependency>
```

再跑一下，随意请求一个url，比如`http://localhost:8080/jms/send?msg=hello`。

好样的~ 没抛啥异常，再去[http://localhost:9411](http://localhost:9411)上看是否有监控记录

![zipkin-query](/img/in-post/learning-zipkin/zipkin-query.png)


Zipkin的Storage component默认是将数据存储在内存中的，所以当你重启Zipkin后数据就会丢失。Zipkin支持的持久化数据存储方案有Cassandra, MySQL和Elasticsearch，请参考[zipkin-dependencies](https://github.com/openzipkin/zipkin-dependencies)里的说明进行配置。

未完待续……


