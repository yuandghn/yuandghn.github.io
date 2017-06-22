---
layout:     post
title:      "A minimum Spring MVC configuration"
subtitle:   ""
date:       2017-06-22 17:24:15
author:     "Echo Yuan"
#header-img: "img/in-post/hello-world/hello-world.gif"
tags:
    - Spring MVC
    - ActiveMQ
---
在本地起一个基于[Spring MVC](http://docs.spring.io/spring/docs/current/spring-framework-reference/htmlsingle/#spring-web)的Web项目的最小化配置，使用[Maven](http://maven.apache.org/)作为项目管理工具。

首先使用Intellij IDEA创建一个maven-archetype-webapp项目，在pom.xml文件中加入Spring MVC的相关依赖，以下是一小片段。
```
<properties>
    <spring.version>4.2.7.RELEASE</spring.version>
</properties>

<dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-webmvc</artifactId>
    <version>${spring.version}</version>
</dependency>
<dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-context</artifactId>
    <version>${spring.version}</version>
</dependency>
```
其实，只加`spring-webmvc`这一个artifact也行，它自身也依赖了其他Spring的组件，但是我们做事就要做的光明正大嘛，把所有需要的组件都写出来，这样以后查问题也一目了然。

然后在web.xml中添加如下内容：
```
<?xml version="1.0" encoding="UTF-8"?>
<web-app xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns="http://java.sun.com/xml/ns/javaee" xmlns:web="http://java.sun.com/xml/ns/javaee/web-app_2_5.xsd" xsi:schemaLocation="http://java.sun.com/xml/ns/javaee http://java.sun.com/xml/ns/javaee/web-app_3_0.xsd" id="WebApp_ID" version="3.0">

    <display-name>hello</display-name>

    <context-param>
        <param-name>contextConfigLocation</param-name>
        <param-value>
            classpath:applicationContext.xml
        </param-value>
    </context-param>

    <listener>
        <listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>
    </listener>

    <servlet>
        <servlet-name>hello</servlet-name>
        <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
        <init-param>
            <param-name>contextConfigLocation</param-name>
            <param-value>classpath:hello-servlet.xml</param-value>
        </init-param>
        <load-on-startup>1</load-on-startup>
    </servlet>
    <servlet-mapping>
        <servlet-name>hello</servlet-name>
        <url-pattern>/</url-pattern>
    </servlet-mapping>
</web-app>

```
applicationContext.xml和hello-servlet.xml都放在src/resources目录下，Maven会把它们编译到WEB-INF/classes下。

以下是applicationContext.xml的内容：
```
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd">


</beans>
```
哈哈~ 空的就行。

以下是hello-servlet.xml的内容：
```
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xmlns:context="http://www.springframework.org/schema/context"
    xsi:schemaLocation="
        http://www.springframework.org/schema/beans
        http://www.springframework.org/schema/beans/spring-beans.xsd
        http://www.springframework.org/schema/context
        http://www.springframework.org/schema/context/spring-context.xsd">

	<context:component-scan base-package="com.echo"/>
</beans>
```
也很简单，就是扫描指定包下的带Annotation的类并自动注册为Bean，默认情况下是这几个Spring提供的Annotation：@Component，@Repository， @Service和@Controller。

接下来写个Controller
```
@Controller
public class HelloController {

    @RequestMapping("/hello")
    public @ResponseBody String test() {
        return "hello, world! This com from spring!";
    }

}
```

配好Tomcat，准备启动吧~

下面说说我遇到的一个问题。

我刚建好这个maven-archetype-webapp项目时并没有加入Spring MVC，而是在拿[ActiveMQ](http://activemq.apache.org/)和[Netty](http://netty.io/)做测试玩。今天想把Netty集成到Spring MVC里玩，于是按照上面的步骤操作，杯了个具的访问页面时报了错。

>
HTTP Status 500 - Handler processing failed; nested exception is java.lang.NoSuchMethodError: org.springframework.core.annotation.AnnotatedElementUtils.findMergedAnnotation(Ljava/lang/reflect/AnnotatedElement;Ljava/lang/Class;)Ljava/lang/annotation/Annotation;

一开始有点儿慌，没细看就先去Google了下，哈哈~ 网上说可能是jar包冲突引起的，但是我`mvn dependency:tree`看了下没啥冲突。于是静下了心来，仔细看看报错信息。

既然是`AnnotatedElementUtils`类里的方法没找到，那就先打开这个类看看吧。快捷键一试，果然找到两个同名的类。

![double-annotated-element-utils](/img/in-post/a-minimum-spring-mvc-configuration/double-annotated-element-utils.png)

坑，原来之前玩的`activemq-all.5.14.0`里也有这样一个类！Scroll from source去详细看一下。

![many-spring-classes-in-activemq-all](/img/in-post/a-minimum-spring-mvc-configuration/many-spring-classes-in-activemq-all.png)

不光这个，还有好多其他Spring的类也在这个里面。

问题现在找到了，那如何解决呢？
1. 将active-mq-all.5.14.0的[源代码](http://archive.apache.org/dist/activemq/)下载下来，用你Spring core中的那个类替换掉active-mq中的这个类，然后重新编译、打包、安装(如果用的Maven的话)。
2. 寻求将当前所用的Spring版本进行降级处理，以匹配active-mq-all.5.14.0中的这个类。毕竟Spring的版本越新，ActiveMQ越难匹配。经过测试，spring-4.1.7.RELEASE可以，spring-4.2.7.RELEASE就不行了，所以就用4.1.7吧。

如果要下个结论的话，个人还是倾向于Spring版本降级的方法，毕竟现在项目都在使用Maven，修改active-mq源代码的方式明显不合适。


