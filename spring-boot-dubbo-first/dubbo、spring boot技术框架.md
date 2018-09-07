# 技术框架

## Spring boot

### Spring注解

#### 通用注解

##### @Autowired

##### @Resource

##### @Component

##### @Service

##### @Scope

##### @Bean

##### @Configuration

##### @ComponentScan

##### @SpringBootApplication

##### @EnableAutoConfiguration

#### JPA注解

##### @Id

##### @Repository

##### @Transient

##### @Basic

##### @Column

##### @GeneratedValue

##### @MappedSuperClass

##### @NoRepositoryBean

##### @Entity、@Table(name="")

##### @SequenceGeneretor

##### @JsonIgnore

##### @JoinColumn

##### @OneToOne

##### @OneToMany

##### @ManyToOne

#### SpringMVC注解

##### @Controller

##### @RestController

##### @ResponseBody

##### @RequestMapping

##### @RequestParam

##### @PathVariable

##### @ControllerAdvice

##### @ExceptionHandler（Excption.class）

### Spring boot核心

#### 基本配置

##### 入口类（Application）

##### @SpringBootApplication注解

###### @SpringBootConfiguration

###### @EnableAutoConfiguration

###### @ComponentScan

##### 关闭特定的自动配置

##### 定制Banner

###### 修改Banner

###### 关闭Banner

##### 全局配置文件

###### application.properties

###### application.yml

##### 适用xml配置

#### 外部配置

#### 日志配置

#### profile配置

#### 运行原理

##### 核心注解

##### @Enable*注解

###### @EnableAspectAutoProxy

###### @EnableAsync

###### @EnableScheduling

###### @EnableWebMVC

###### @EnableConfigurationProperties

###### @EnableJpaRepositories

###### @EnableTransactionManagement

###### @EnableCaching

### Starter模块

#### Spring-boot-starter

#### Spring-boot-starter-actuator

#### Spring-boot-starter-amqp

#### Spring-boot-starter-aop

#### Spring-boot-starter-artemis

#### Spring-boot-starter-batch

#### Spring-boot-starter-cache

#### Spring-boot-starter-cloud-connectors

#### Spring-boot-starter-data-elasticsearch

#### Spring-boot-starter-data-gemfire

#### Spring-boot-starter-data-jpa

#### Spring-boot-starter-data-mongodb

#### Spring-boot-starter-data-rest

#### Spring-boot-starter-data-solr

#### Spring-boot-starter-groovy-templates

#### Spring-boot-starter--hateoas

#### Spring-boot-starter-hornetq

#### Spring-boot-starter-jdbc

#### Spring-boot-starter-integration

#### Spring-boot-starter-jersey

#### Spring-boot-starter-jta-atomikos

#### Spring-boot-starter-jta-bitronix

#### Spring-boot-starter-mail

#### Spring-boot-starter-mobile

#### Spring-boot-starter-mustache

#### Spring-boot-starter-redis

#### Spring-boot-starter-security

#### Spring-boot-starter-social-facebook

#### Spring-boot-starter-linkedin

#### Spring-boot-starter-social-twitter

#### Spring-boot-starter-test

#### Spring-boot-starter-thymeleaf

#### Spring-boot-starter-freemarker

#### Spring-boot-starter-velocity

#### Spring-boot-starter-web

#### Spring-boot-starter-websocket

#### Spring-boot-starter-ws

### Web开发

#### Web相关配置

##### 自动配置ViewResolver

##### 自动配置静态资源

###### 类路径文件

###### webjar

##### 自动配置Formatter和Converter

##### 自动配置HttpMessageConverters

###### JacksonHttpMessageConfiguration

###### GsonHttpMessageConvertersConfiguration

##### 注册Servlet、Filter、Listener

#### 内嵌servlet容器配置

##### 公用配置

###### server.port

###### server.session-timeout

###### server.context-path

##### tomcat配置

###### server.tomcat.uri-encoding

###### server.tomcat.cpmpression

##### 替换tomcat

###### jetty

###### undertow

#### Favicon配置

##### 关闭favicon

##### 设置favicon

#### WebSocket

##### 广播式

##### 点对点式

### 数据访问

#### Spring Data Jpa

#### Spring Data Rest

##### 继承方式

##### @Import方式

#### 声明式事物

#### 数据缓存

##### CacheManager

##### 声明式缓存注解

###### @Cacheable

###### @CachePut

###### @CacheEvict

###### @Caching

##### 开启声明式缓存

#### MongoDB

##### 映射注解

##### MongoTemplate

###### @Document

###### @ID

###### @DbRef

###### @Field

###### @Version

##### Repository支持

### 部署与测试

#### 开发热部署

##### 模板热部署

###### freemarker

###### thymeleaf

###### groovy

###### velocity

##### Spring Loaded

##### spring-boot-devtools

#### 部署方式

##### jar

###### 打包

###### 运行

###### 注册为Linux服务

##### war

###### 打包

### 应用监控

#### 端点

##### trace

##### shutdown

##### mapping

##### metrics

##### info

##### autoconfig

##### env

##### dump

##### configprops

##### beans

##### actuator

##### health

#### 定制端点

##### 修改端点

##### 开启端点

##### 关闭端点

##### 开启所需端点

##### 定制端点访问路径

##### 定制端点访问端口

##### 关闭http端点

#### 自定义端点

## Dubbo

### Framework

#### RPC

##### Protocol

##### Monitor

##### Cluster

##### Registry

##### Proxy

##### Config

#### Business

##### Service

#### Remoting

##### Transport

##### Serialize

### dubbo-spring-boot-starter

#### 如何发布dubbo服务

##### 添加依赖

###### <dependency>
	    <groupId>com.alibaba.spring.boot</groupId>
	    <artifactId>dubbo-spring-boot-starter</artifactId>
	    <version>2.0.0</version>
	</dependency>

##### 在application.properties添加dubbo相关配置

###### spring.application.name=dubbo-spring-boot-starter
spring.dubbo.server=true
spring.dubbo.registry=N/A

##### 在Spring Boot Application添加@EnableDubboConfiguration

###### 此标签表明开启dubbo功能

####### @SpringBootApplication
@EnableDubboConfiguration
public class DubboProviderLauncher {
  //...
}

##### 编写dubbo服务

###### 添加@Service（import com.alibaba.config.annotation.Service）注解

####### @Service(interfaceClass = IHelloService.class)
@Componentpublic class HelloServiceImpl implements IHelloService {
  //...
}

#### 如何消费dubbo服务

##### 添加依赖

###### <dependency>
	    <groupId>com.alibaba.spring.boot</groupId>
	    <artifactId>dubbo-spring-boot-starter</artifactId>
	    <version>2.0.0</version>
	</dependency>

##### 添加dubbo相关配置信息

###### 样例：在application.properties中添加

####### spring.application.name=dubbo-spring-boot-starter

##### 开启@EnableDubboConfiguration

###### @SpringBootApplication
@EnableDubboConfiguration
public class DubboConsumerLauncher {
  //...
}

##### 通过@Reference注入需要使用的interface

###### @Componentpublic class HelloConsumer {
  @Reference(url = "dubbo://127.0.0.1:20880")
  private IHelloService iHelloService;
  
}

### Dubbo扩展组件

#### 注册

##### registryFactory

###### dubbo

###### multicast

###### redis

###### zookeeper

##### configuratorFactory

###### override

###### absent

##### zookeeperTransporter

###### curator

###### zkclient

#### 协议

##### protocol

###### dubbo

###### injvm

###### memcached

###### hession

###### thrift

###### rmi

###### webservice

###### redis

###### registry

###### filter(protocolFilterWrapper)

###### listener(protocolListenerWrapper)

##### serialization

###### compactedjava

###### hession

###### dubbo

###### fastjson

###### json

###### nativeJava

##### codec2

###### transport

###### telnet

###### exchange

###### dubbo

#### 代理

##### proxyFactory

###### stub(wrapper)

###### jdk

###### javassist

##### compiler

###### jdk

###### javassist

###### Adaptive

#### 请求拦截

##### filter

###### activelimit

###### classloader

###### cache

###### compatible

###### future

###### mock

###### token

###### timeout

###### monitor

###### tpslimit

###### trace

###### valdation

###### Generic

###### ExecuteLimit

###### Exception

###### echo

###### consumerContext

###### context

###### accesslog

#### 容器

##### container

###### spring

###### registry

###### jetty

###### logback

###### log4j

##### extensionFactory

###### spring

###### spi

###### Adaptive(adaptive)

#### 监控

##### monitorFactory

###### dubbo

##### telnetHandler

#### 通信

##### exchanger

###### header

##### transporter

###### netty

###### mina

###### grizzly

##### httpBinder

###### jetty

###### servlet

#### 事件处理

##### threadpool

###### limited

###### cached

###### fixed

##### dispatcher

###### all

###### connectionOrdered

###### messageOnly

###### execution

###### direct

#### 容错

##### cluster

###### mock

###### failover

###### failfast

###### failsafe

###### failback

###### forking

###### avaliable

###### mergable

###### broadcast

#### 路由

##### loadbalance

###### consistentHash

###### roundRobin

###### randomLoad

###### leastActive

##### routerFactory

###### condition

###### file

###### script

#### 辅助

##### cacheFactory

###### jcache

###### threadlocal

###### lru

##### validation

###### jvalidation

##### merger

###### map

###### set

###### list

###### byte

###### char

###### short

###### int

###### long

###### float

###### double

###### boolean

## Zookeeper

## Redis
