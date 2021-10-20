---
layout:     post
title:      "Deep into Spring Boot Actuator Health Endpoint"
subtitle:   ""
date:       2021-09-28 00:19:00
author:     "Echo Yuan"
tags:
    - Spring Boot Actuator
    - Health Endpoint
---

Spring Boot Actuator现在已经几乎成为了应用的标配模块，只要依赖了它，应用就会自动加持`/actuator/health`，然后就可以交给容器环境来探测了。这一切看起来似乎是很简单很自然的事情，可你真的对它了解吗？本文就试图对这个Health Endpoint做一些深入的剖析。

事情的起因也是日常工作中常见的问题套路。远程环境的某个服务之前一直好好的，最近却经常被k8s重启，排查后发现是`/actuator/health`超时了，而本地开发环境却依然秒回。接着挖，发现是应用要发邮件，所以依赖了`spring-boot-starter-mail`，而mail的测试账号是临时申请的126邮箱，跨国的网络通信导致`/actuator/health`不能如期返回。吃一堑、长一智，由表及里地对Actuator多一些了解也就提上了日程。

本文使用的Spring Boot版本为2.3.9.RELEASE。

### Endpoints
先贴一段[官方文档](https://docs.spring.io/spring-boot/docs/2.3.9.RELEASE/reference/html/production-ready-features.html#production-ready)的描述
> Actuator endpoints let you monitor and interact with your application. Spring Boot includes a number of built-in endpoints and lets you add your own. For example, the `health` endpoint provides basic application health information.

> Each individual endpoint can be [enabled or disabled](https://docs.spring.io/spring-boot/docs/2.3.9.RELEASE/reference/html/production-ready-features.html#production-ready-endpoints-enabling-endpoints) and [exposed (made remotely accessible) over HTTP or JMX](exposed (made remotely accessible) over HTTP or JMX). An endpoint is considered to be available when it is both enabled and exposed. The built-in endpoints will only be auto-configured when they are available. Most applications choose exposure via HTTP, where the ID of the endpoint along with a prefix of `/actuator `is mapped to a URL. For example, by default, the `health` endpoint is mapped to `/actuator/health`.

这段描述提纲挈领，使我们了解了Endpoint的关键操作：是否启用->是否暴露 & 暴露方式->如何触达。

通过HTTP暴露的Endpoints默认只开了两个，`/actuator/health`和`/actuator/info`，所以`/actuator/beans`、`/actuator/env`之类的默认是不可触达的。本文不涉及JMX的暴露方式。

想必你也猜到了，Spring Boot Actuator一定是通过某个配置项来做到的。正如你所想，以下引自[官方文档](https://docs.spring.io/spring-boot/docs/2.3.9.RELEASE/reference/html/appendix-application-properties.html#common-application-properties-actuator)：

| Key | Default Value | Description |
|------|------|------|
|management.endpoints.web.exposure.include|[health, info]|Endpoint IDs that should be included or '*' for all.|

### Web Endpoints
通过HTTP方式暴露的Endpoints称作Web Endpoints，那都有哪些内置的Web Endpoints呢？Actuator贴心地提供了一个`discovery page`来告诉你，默认情况下你可以通过`{contextPath}/actuator`来查看。

```javascript
{
  "_links": {
    "self": {
      "href": "http://localhost:8080/demo/actuator",
      "templated": false
    },
    "beans": {
      "href": "http://localhost:8080/demo/actuator/beans",
      "templated": false
    },
    "caches": {
      "href": "http://localhost:8080/demo/actuator/caches",
      "templated": false
    },
    ...
    "health-path": {
		"href": "http://localhost:39001/demo/actuator/health/{*path}",
		"templated": true
	},
	"health": {
		"href": "http://localhost:39001/demo/actuator/health",
		"templated": false
	},
    ...
  }
}
```

### Health Information
Health Information是由Health这个Endpoint提供的，它也是本文重点讲述的对象。它的结果是对应用里所有已注册的HealthIndicators的结果的聚合，那既然是聚合的结果，可能会花费一些时间才能获得。

Health Endpoint返回的信息繁简与否，取决于以下两个属性的配置

| Key | Default Value | Description |
|------|------|------|
|management.endpoint.health.show-components||When to show components. If not specified the 'show-details' setting will be used.|
|management.endpoint.health.show-details|never|When to show full health details.|

`show-details`属性的取值如下

| Name | Description |
|------|------|
| never |Details are never shown.|
|when-authorized|Details are only shown to authorized users. Authorized roles can be configured using `management.endpoint.health.roles`.|
| always |Details are shown to all users.|

我们将`management.endpoint.health.show-details`设置为`always `。目前demo项目里只引入了`spring-boot-starter-amqp`的依赖，所以`/demo/actuator/health`的返回如下

```javascript
{
  "status": "UP",
  "components": {
    "diskSpace": {
      "status": "UP",
      "details": {
        "total": 250790436864,
        "free": 62605455360,
        "threshold": 10485760,
        "exists": true
      }
    },
    "ping": {
      "status": "UP"
    },
    "rabbit": {
      "status": "UP",
      "details": {
        "version": "3.8.9"
      }
    }
  }
}
```
可以看到除了rabbit，还有diskSpace和ping这两个HealthIndicator。下面我们就来自定义一个`HealthIndicator`。

### Custom HealthIndicator
写一个自定义的`HealthIndicator`相当的简单，只要注册一个Spring Bean并让它实现[HealthIndicator](https://github.com/spring-projects/spring-boot/tree/v2.3.9.RELEASE/spring-boot-project/spring-boot-actuator/src/main/java/org/springframework/boot/actuate/health/HealthIndicator.java)接口即可。

```java
@Component
public class RandomHealthIndicator implements HealthIndicator {

    @Override
    public Health health() {
        double chance = ThreadLocalRandom.current().nextDouble();
        Health.Builder status = Health.up();
        if (chance > 0.9) {
            status = Health.down();
        }
        try {
            // 假装做了很多事
            TimeUnit.SECONDS.sleep(1L);
        } catch (InterruptedException ignore) {
        }
        return status.withDetail("chance", chance)
                .withDetail("strategy", "thread-local")
                .build();
    }
}
```

这时再看看`/demo/actuator/health`的返回

```javascript
{
  "status": "UP",
  "components": {
    "diskSpace": {
      "status": "UP",
      "details": {
        "total": 250790436864,
        "free": 62561599488,
        "threshold": 10485760,
        "exists": true
      }
    },
    "ping": {
      "status": "UP"
    },
    "rabbit": {
      "status": "UP",
      "details": {
        "version": "3.8.9"
      }
    },
    "random": {
      "status": "UP",
      "details": {
        "chance": 0.21226285141894985,
        "strategy": "thread-local"
      }
    }
  }
}
```
如期出现。也可以访问某个具体的indicator，比如我们刚刚创建的`/demo/actuator/health/random`

```javascript
{
  "status": "UP",
  "details": {
    "chance": 0.012398098477797048,
    "strategy": "thread-local"
  }
}
```
那这个名字`random`是怎么确定的呢？默认是先取bean name，然后裁掉`HealthIndicator`后缀得到的。所以，如果我们自定义了bean name，如@Component("rand")，那么就要这么访问`/demo/actuator/health/rand`了。


### Health Status
`Health Status`可以表示各个components的健康状况以及由所有的components聚合而来的代表整个系统的健康状况。

框架预定义了以下几个`Health Status`和其对应的HTTP status code

| Status | Mapping |
|------|------|
| DOWN |SERVICE_UNAVAILABLE (503)|
| OUT\_OF\_SERVICE |SERVICE_UNAVAILABLE (503)|
| UP |No mapping by default, so http status is 200|
| UNKNOWN |No mapping by default, so http status is 200|

这些`Health Status`是以[public static final](https://github.com/spring-projects/spring-boot/blob/v2.3.9.RELEASE/spring-boot-project/spring-boot-actuator/src/main/java/org/springframework/boot/actuate/health/Status.java)的形式来声明的，并不是用的Java enums，所以你可以定义自己的`Health Status`

```java
Health.Builder warning = Health.status("FATAL");
```

`Health Status`会影响Health Endpoint的HTTP status code的输出，在没有为自定义的`FATAL`status设置对应的HTTP status code时，HTTP status code会是`200`。可通过配置的方式进行设置(大小写不敏感)

```
management.endpoint.health.status.http-mapping.fatal=503
```
也可以定义一个`HttpCodeStatusMapper`类型的bean来做下映射。

单只配了http-mapping还不够，还要记得去调整一下`Health Status`严重程度的顺序，不然你的FATAL在Health Endpoint里仍然会被认作`200`的HTTP status code来返回，当存在任意一个`UP`的情况下。

```
management.endpoint.health.status.order=fatal,down,out-of-service,unknown,up
```
Actuator的逻辑就是先把各个components的status收集起来，然后在这个status order列表里寻找下标最小的那个status，并将其http-mapping作为最终的HTTP status code。具体可参考`SimpleStatusAggregator`的源码。

### Measure Health Endpoint
前面多次提到，Health Endpoint的结果是多个components的结果的聚合，所以它所花费的时间也是这些components顺序执行完所需时间之和。

在Kubernetes环境中，如果你的应用依赖了较多的components，且它们之中有的health check需要走网络，比如跨国/跨地区的mail服务，那么Health Endpoint有可能会超出容器探针所配置的策略值，进而影响应用的启动和运行。

作为实验目的，我们现在希望能对所有的components做一下耗时测量。

经过一番对源码的探索，我们找到了Health Endpoint的扩展点所在

```java
@Component
public class CustomHealthEndpointWebExtension extends HealthEndpointWebExtension {

    public CustomHealthEndpointWebExtension(HealthContributorRegistry registry, HealthEndpointGroups groups) {
        super(registry, groups);
    }

    @Override
    protected HealthComponent getHealth(HealthContributor contributor, boolean includeDetails) {
        Instant start = Instant.now();
        HealthComponent healthComponent = super.getHealth(contributor, includeDetails);
        Duration interval = Duration.between(start, Instant.now());
        if (healthComponent instanceof Health) {
            Health health = (Health) healthComponent;
            return new Health.Builder(health.getStatus(), health.getDetails())
                    .withDetail("timeElapsedInMillis", interval.toMillis())
                    .build();
        }
        return healthComponent;
    }

    @Override
    protected HealthComponent aggregateContributions(ApiVersion apiVersion, Map<String, HealthComponent> contributions, StatusAggregator statusAggregator, boolean showComponents, Set<String> groupNames) {
        // 可惜目前无法对aggregate的HealthComponent做加强操作
        return super.aggregateContributions(apiVersion, contributions, statusAggregator, showComponents, groupNames);
    }

}
```

代码本身比较简单，这里想着重说一下里面的HealthComponent。以下是关于它的层次结构
![health-component-hierarchy](/img/in-post/spring-boos-actuator/health-component-hierarchy.png)
HealthComponent有三种：`Health`、`CompositeHealth`、`SystemHealth`。

通过`implements HealthIndicator`自定义的health indicator或框架内置的那些通过`extends AbstractHealthIndicator`的health indicator，返回的都是Health；包含多个components的indicator，比如自定义了[group](https://docs.spring.io/spring-boot/docs/2.3.9.RELEASE/reference/html/production-ready-features.html#production-ready-health-groups)`/actuator/health/mygroup`，返回的就是CompositeHealth；而`SystemHealth`则只有`/actuator/health`的时候才会返回。

看一下我们测量的效果

```javascript
{
  "status": "UP",
  "components": {
    "diskSpace": {
      "status": "UP",
      "details": {
        "total": 250790436864,
        "free": 60874444800,
        "threshold": 10485760,
        "exists": true,
        "timeElapsedInMillis": 0
      }
    },
    "ping": {
      "status": "UP",
      "details": {
        "timeElapsedInMillis": 0
      }
    },
    "rabbit": {
      "status": "UP",
      "details": {
        "version": "3.8.9",
        "timeElapsedInMillis": 90
      }
    },
    "random": {
      "status": "UP",
      "details": {
        "chance": 0.15429430172284797,
        "strategy": "thread-local",
        "timeElapsedInMillis": 1005
      }
    }
  }
}
```

遗憾的是，目前还没有办法将总耗时汇总到和`components`平级的位置，这在`aggregateContributions`方法中也有注释。

### Auto-configured HealthIndicators
我们不准备在这里列出所有内置的HealthIndicators，只是简单看一下它们的共同父类`AbstractHealthIndicator`的核心定义

```java
public abstract class AbstractHealthIndicator implements HealthIndicator {
	
	@Override
	public final Health health() {
		Health.Builder builder = new Health.Builder();
		try {
			doHealthCheck(builder);
		}
		catch (Exception ex) {
			if (this.logger.isWarnEnabled()) {
				String message = this.healthCheckFailedMessage.apply(ex);
				this.logger.warn(StringUtils.hasText(message) ? message : DEFAULT_MESSAGE, ex);
			}
			builder.down(ex);
		}
		return builder.build();
	}

	/**
	 * Actual health check logic.
	 * @param builder the {@link Builder} to report health status and details
	 * @throws Exception any {@link Exception} that should create a {@link Status#DOWN}
	 * system status.
	 */
	protected abstract void doHealthCheck(Health.Builder builder) throws Exception;

}	
```
它封装了Health实例的创建和异常处理，相比之下，我们前面定义的`RandomHealthIndicator`表现的就有点儿差了。

最后再来看一下MailHealthIndicator，也就是那个让我们的应用超时的不安分因子

```java
/**
 * {@link HealthIndicator} for configured smtp server(s).
 *
 * @author Johannes Edmeier
 * @since 2.0.0
 */
public class MailHealthIndicator extends AbstractHealthIndicator {

	private final JavaMailSenderImpl mailSender;

	public MailHealthIndicator(JavaMailSenderImpl mailSender) {
		super("Mail health check failed");
		this.mailSender = mailSender;
	}

	@Override
	protected void doHealthCheck(Builder builder) throws Exception {
		builder.withDetail("location", this.mailSender.getHost() + ":" + this.mailSender.getPort());
		this.mailSender.testConnection();
		builder.up();
	}

}
```

`this.mailSender.testConnection()`是真的会走下网络连接。

### Kubernetes Probes
为了让应用更适应容器化环境，Spring Boot从2.3起提供了开箱即用的[探针技术](https://github.com/spring-projects/spring-boot/wiki/Spring-Boot-2.3-Release-Notes#liveness-and-readiness-probes)来暴露应用程序的可用性状态([Application Availability State](https://docs.spring.io/spring-boot/docs/2.3.9.RELEASE/reference/html/spring-boot-features.html#boot-features-application-availability))。

如果你的应用部署在Kubernetes环境中，那么这两个Health Groups，`/actuator/health/liveness`和`/actuator/health/readiness`，将自动变为可用，其背后是`LivenessStateHealthIndicator`和`ReadinessStateHealthIndicator`在提供支撑。

> In general, the "Liveness" state should not be based on external checks, such as `Health checks`. If it did, a failing external system (a database, a Web API, an external cache) would trigger massive restarts and cascading failures across the platform.

依据官方的说明，使用`/actuator/health`作为probe并不是一个好的实践，前面我们也有讲它可能会变成一个较为耗时的操作，且受网络波动的影响你的应用也会变得时好时坏、难以捉摸。

如果你仍然想继续用`/actuator/health`或者因为Spring Boot版本小于2.3.9，那么把`management.health.defaults.enabled`设置为`false`可能会是个好的选择，以此来保证`/actuator/health`的网络联通性但又不会去做任何实际的health check操作，这其实和你自己写一个空的`handler method`没什么区别。

如果你还是想让它做一些实际的health check操作，但又希望把其中一些检查项排除在外，可以单独设置这些项的启用，比如`management.health.mail.enabled: false`。

`management.health.defaults.enabled: false`会使所有内置的Health Contributors不再自动注册为Spring beans，也就直接导致了`/actuator/health`无兵(bean)可用。它是通过`ConditionalOnEnabledHealthIndicator`注解在auto configuration阶段做到的，具体可参考一下`DataSourceHealthContributorAutoConfiguration`的源码。

如果你自定义了Health Indicator，那最好也给它加上`ConditionalOnEnabledHealthIndicator`注解，不然它就脱离`management.health.defaults.enabled: false`的控制了，这个[Unit Test](https://github.com/spring-projects/spring-boot/blob/2.3.x/spring-boot-project/spring-boot-actuator-autoconfigure/src/test/java/org/springframework/boot/actuate/autoconfigure/health/HealthContributorAutoConfigurationTests.java)对此有所演示。

以下是三个快速的Q&A

##### Q1 - 应用是在本地直接起的，怎样可以使这两个Health Groups也可用？
设置`management.endpoint.health.probes.enabled`为`true`
##### Q2 - 应用是怎样auto-detects到身处Kubernetes环境中的？
其实很简单，文档请参考[cloud-deployment-kubernetes](https://docs.spring.io/spring-boot/docs/2.3.9.RELEASE/reference/html/deployment.html#cloud-deployment-kubernetes)，代码请参考`AvailabilityProbesAutoConfiguration`，就是在这里决定是否创建`livenessStateHealthIndicator`、`readinessStateHealthIndicator`以及两个Health Groups`liveness`、`readiness`，也是在这个地方(ProbesCondition)判断当前环境是不是Kubernetes的。
##### Q3 - 如何主动改变Application Availability State？
在你的程序里可以使用`AvailabilityChangeEvent.publish`这个便捷的静态方法来发布新的availability state，只要拿到`ApplicationEventPublisher`Bean就行。

### JMX Tips
如果你用Intellij IDEA在本地调试的过程中发现JMX Endpoint不知怎地被启用了，那可能是IDEA的锅
![intellij-idea-enable-JMX-agent](/img/in-post/spring-boos-actuator/intellij-idea-enable-JMX-agent.png)

### 结语
Actuator虽然看起来简单，可能就是依赖一引的事儿，不过如果能多了解一点儿其内部的工作原理，对日常工作应该也是有帮助的。

### 参考文档
[https://www.baeldung.com/spring-boot-health-indicators](https://www.baeldung.com/spring-boot-health-indicators)

[https://www.baeldung.com/spring-boot-actuators](https://www.baeldung.com/spring-boot-actuators)

[https://docs.spring.io/spring-boot/docs/2.3.9.RELEASE/reference/html/production-ready-features.html#production-ready](https://docs.spring.io/spring-boot/docs/2.3.9.RELEASE/reference/html/production-ready-features.html#production-ready)

[https://docs.spring.io/spring-boot/docs/2.3.9.RELEASE/reference/html/appendix-application-properties.html#common-application-properties-actuator](https://docs.spring.io/spring-boot/docs/2.3.9.RELEASE/reference/html/appendix-application-properties.html#common-application-properties-actuator)

[https://docs.spring.io/spring-boot/docs/2.3.9.RELEASE/reference/html/spring-boot-features.html#boot-features-application-availability](https://docs.spring.io/spring-boot/docs/2.3.9.RELEASE/reference/html/spring-boot-features.html#boot-features-application-availability)

[https://spring.io/blog/2020/03/25/liveness-and-readiness-probes-with-spring-boot](https://spring.io/blog/2020/03/25/liveness-and-readiness-probes-with-spring-boot)