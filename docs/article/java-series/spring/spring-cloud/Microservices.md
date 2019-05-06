# 概述

## 场景介绍

### 注册

- Eureka Client：负责将这个服务的信息注册到Eureka Server中  
- Eureka Server：注册中心，里面有一个注册表，保存了各个服务所在的机器和端口号  

在应用启动时，Eureka客户端向服务端注册自己的服务信息？  
同时将服务端的服务信息缓存到本地。客户端会和服务端周期性的进行心跳交互，以更新服务租约和服务信息？  
注册中心要最先启动，疑问注册中心是动态变化的吗？  

通常配合着 Feign。  

Feign遇到异常的时候呢？  

## 负载均衡 Ribbon

Ribbon是一个基于HTTP和TCP的客户端负载均衡工具，它基于Netflix Ribbon实现。通过Spring Cloud的封装，可以让我们轻松地将面向服务的REST模版请求自动转换成客户端负载均衡的服务调用。

通常Ribbon是和Feign以及Eureka紧密协作：  

- 首先Ribbon会从 Eureka Client里获取到对应的服务注册表，也就知道了所有的服务都部署在了哪些机器上，在监听哪些端口号。
- 然后Ribbon就可以使用默认的Round Robin算法，从中选择一台机器

Feign就会针对这台机器，构造并发起请求。  

## 隔离、熔断以及降级 Hystrix

隔离：让独立的服务之间不因调用顺序而相互影响，有点像多线程  
熔断：直接取消掉某个挂掉的服务  
降级：可以对某个挂掉的服务做记录，以便事后处理的需要  

服务与服务之间的依赖性，故障会传播，会对整个微服务系统造成灾难性的严重后果，这就是服务故障的“雪崩”效应。

## 网关 Zuul

像android、ios、pc前端、微信小程序、H5等等，不用去关心后端有几百个服务，就知道有一个网关，所有请求都往网关走，网关会根据请求中的一些特征，将请求转发给后端的各个服务。

有一个网关之后，还有很多好处，比如可以做统一的降级、限流、认证授权、安全，等等。

## 参考

http://www.infoq.com/cn/news/2016/08/Monomer-architecture-Micro-servi 从单体架构迁移到微服务，8个关键的思考、实践和经验
https://en.wikipedia.org/wiki/Microservices 基础概念和理论
微服务实际上和分库、分表并无太多关系。分布式应用和分布式数据库是两个层面、两个概念。

https://projects.spring.io/spring-cloud/
https://docs.spring.io/spring-boot/docs/current/reference/htmlsingle/ 
http://www.infoq.com/cn/articles/basis-frameworkto-implement-micro-service 实施微服务，我们需要哪些框架
http://www.infoq.com/cn/articles/boot-microservices 使用SpringBoot创建微服务

微服务使得服务便于自治、便于协作。割据总会带来性能上的代价。中央集权和地方自治总是需要不断磨合，然后才能愈加协调。