---
layout: post
title:  springCloud拾遗1-可抹去的注解@EnableDiscoveryClient
excerpt: "springCloud版本变化很快，对于很多基于此框架开发的搬砖党来说可能真的很难受，一个版本还没研究透就又刷新版本，而老版本意味着经典的同时也意味着即将成为时代的眼泪，但是路总是一步步走出来的，戒骄戒躁，记录每一个新发现"
categories: [java]
tags: [springCloud]
comments: true
---

#### @EnableDiscoveryClient&@EnableEurekaClient

上面俩个注解相信很多人都不会很陌生，在使用eureka或其组建作为注册中心时，可以使用@EnableDiscoveryClient，如果用的注册中心时eureka同时还可以使用@EnableEurekaClient，但是一次偶然的机会，我忘记加该注解了，结果意外的发现照样可以注册上，于是，这篇博文就来了

先来回顾一下注解的使用，约定由于配置，万变不离其宗

##### 1）pom引入依赖

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
</dependency>
```

##### 2）主启动类加上注解

```java
@SpringBootApplication
@EnableEurekaClient
@MapperScan("com.guwukeji.varysucenter.dao")
@EnableCaching
public class VarysUcenterApplication extends SpringBootServletInitializer {

	@Override
	protected SpringApplicationBuilder configure(SpringApplicationBuilder application) {
		return application.sources(VarysUcenterApplication.class);
	}

	public static void main(String[] args) {
		SpringApplication.run(VarysUcenterApplication.class, args);
	}
}
```

##### 3)application.yml加上配置项

```yml
eureka:
  client:
    serviceUrl:
      defaultZone: http://varys:varys12345@172.16.60.8:8761/eureka/
      #defaultZone: http://varys:varys12345@localhost:8761/eureka/
  instance:
    hostname: ${spring.cloud.client.ip-address}
    prefer-ip-address: true
    instance-id: ${spring.cloud.client.ip-address}:${spring.application.name}:${server.port}
```

***那么来交流一下，为什么不加注解也是可以的呢？***

**从Spring Cloud Edgware开始，`@EnableDiscoveryClient` 或`@EnableEurekaClient` 可省略。只需加上相关依赖，并进行相应配置，即可将微服务注册到服务发现组件上。**

***分析一下：但Spring Cloud为什么要这么设计/改进呢？***

这是由于在实际项目中，我们可能希望实现“不同环境不同配置”的效果——例如：在开发环境中，不注册到Eureka Server上，而是服务提供者、服务消费者直连，便于调测；在生产环境中，我们又希望能够享受服务发现的优势——服务消费者无需知道服务提供者的绝对地址。为适应该需求，Spring Cloud Commons进行了改进，相关Issue：<https://github.com/spring-cloud/spring-cloud-commons/issues/218> 。

**如不想将服务注册到Eureka Server**，只需在application*.yml设置

```java
spring.cloud.service-registry.auto-registration.enabled=false 
```

或者在主启动类加上注解

```java
@EnableDiscoveryClient(autoRegister = false)
```

---

摘自文章：[Spring Cloud Edgware新特性之七：可选的EnableDiscoveryClient注解](http://www.itmuch.com/spring-cloud/edgware-new-optional-enable-discovery-client/)

