---
layout: post
title: springcloud-config 使用
---
springcloud-config是springcloud提供的统一配置管理中心，用来管理微服务中大量存在的配置文件，能够做到集中管理和动态更新。

## springboot配置优先级顺序

1. 命令行
2. java 系统变量
3. 操作系统变量
4. springcloud config
5. application-{profile}.yml
6. application.yml
7. bootstrap.yml
8. @configuration配置类

## 使用springcloudconfig原因

- 配置文件过多修改起来麻烦
- 很多配置文件都在使用同样的配置，重复配置导致资源浪费
- 修改配置需要重启线上应用，soringcloudconfig可以做到动态修改配置

## 配置configserver

引入springcloudconfig

```java
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-config</artifactId>
</dependency>
<dependency>
	<groupId>org.springframework.cloud</groupId>
	<artifactId>spring-cloud-config-server</artifactId>
</dependency>
```

修改启动类，添加@EnableConfigServer注解

```java
@SpringBootApplication
@EnableConfigServer
public class ScConfigServerApplication {

    public static void main(String[] args) {
        SpringApplication.run(ScConfigServerApplication.class, args);
    }
}
```

配置git仓库和用户名密码

```java
spring:
  cloud:
    config:
      server:
        git:
          uri: git@gitlab.huixiaoer.com:java-micro-service/sc-config-repo.git
          search-paths: qiniuyun,datasource,application
          username: xxxx
          password: xxx
          label: master
  application:
    name: sc-config-server
    
server:
  port: 9301
```

git配置仓库设计

很多配置都是多服务需要的，如果在每个服务的配置文件里都写一份的话会很难修改和维护,所以需要对现有配置进行整理，目前设计如下

![1543818712437](/home/lifei/Pictures/1543818712437.png)

- 其中datasource和qiniuyun等都是多个服务需要使用的配置，所以把他们分离出来，并且配置不同环境需要使用的配置。
- 如果需要分离出来更多的配置的话，需要在spring.cloud.config.server.git.search-paths里加入需要的目录。

## configclient配置

引入config

```java
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-config</artifactId>
</dependency>
```

配置configserver

需要注意的是configserver的配置不能写在application.yml里，这样的话会不生效，需要新建一个bootstrap.yml文件，这个文件主要加载springboot启动的配置，加载顺序优先级最高。

```java
spring:
  application:
    name: ms-customer-user
  cloud:
    config:
      label: master
      profile: dev
      name: ms-customer-user, qiniuyun, datasource
      uri: http://localhost:9301/
```

选择要使用的配置

- 首先设置application.name，应用会在configserver上搜索{application.name}-{profile}.yml|properties文件
- 如果还需要{application.name}-{profile}.yml之外的其他配置，需要在spring.cloud.name里添加需要的配置文件前缀,例如datasource-{profile}.yml

## 其他

### 加入configserer之后如何在本地调试代码？

有以下几种选择

- 本地启动eurekaserver和configserver，再启动应用进行调试。

- 本地启动configserver，再启动应用，如果只是单独项目的调试并不需要eureka，可以再pom里需要common包的依赖，如下就不会包eurekaserver 不存在的错误。

  ```
  <dependency>
      <groupId>com.huixiaoer</groupId>
      <artifactId>common</artifactId>
      <version>1.0-SNAPSHOT</version>
      <exclusions>
          <exclusion>
              <groupId>org.springframework.cloud</groupId>
              <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
          </exclusion>
      </exclusions>
  </dependency>
  ```

- 只需要启动应用就行，这样的话需要保留之前的配置文件，根据前边提到的配置优先级，springcloudconfig的顺序高于本地配置，所以修改本地配置并不会对线上或者测试环境配置有影响。不过这样在使用configserver之后需要保留之前的配置方法。

### 配置的使用安全问题

使用gitlab当作配置仓库对项目的可用性有没有影响？

- gitlab相对单个数据库是更加不容易挂掉的，但是数据库可以通过集群配置高可用，gitlab就很难实现
- gitlab如果在configserver启动之后挂掉的话，并不会对服务使用配置文件造成大的影响，因为git仓库中的配置已经clone到了服务器本地仓库，但是如果要更新配置会出现麻烦
- config使用数据库作为仓库也很简单

git仓库存放配置是否安全？

- 私有项目需要git权限验证，已经比较安全了，但是相对数据库安全还是相对不可控的。
- 极端情况下，gitlab服务器被入侵会是配置不安全。

configserver有没有必要加上权限认证？

- 如果是测试或者本地的话，肯定是不需要验证比较方便。
- 线上部署的话，在集群中已经有很多安全限制了，所以我觉得还是不需要配置

## 目前项目配置整理

服务应用配置{application-name}-{profile}.yml

这里的appication有ms-internal-user、ms-provider-user、ms-customer-user、ms-cs-case、ms-cs-hotel-bid

### bootstrap.yml

拿这五个项目举例，项目中需要有一个配置文件bootstrap.yml是必需的，其中配置了application-name,configserver地址，端口号，监测端口号，所以bootstrap.yml包含以下内容

```java
spring:
  application:
    name: ms-customer-user
  cloud:
    config:
      label: master
      profile: dev
      name: ms-customer-user, qiniuyun, datasource
      uri: http://localhost:9301/


server:
  port: 9204
management:
  server:
    port: 9504
```

### {application-name}-{profile}.yml

这些文件放在application文件夹下，用来配置一些项目私有的或者跟其他项目不一样的配置，由于mybatis的配置中model的位置有一些不同，所以就可以放在这个配置文件里

```java
mybatis:
  mapper-locations: classpath:mapper/*Mapper.xml
  type-aliases-package: com.huixiaoer.common.model.cs
  check-config-location: false
  configuration:
    logImpl: org.apache.ibatis.logging.log4j2.Log4j2Impl
```

### {source}-{profile}.yml

这些配置是从各个项目中重复内容分离出来的配置，包括数据库、redis、七牛云、im云信、阿里云短信等等，单独拿出来之后加上版本信息可以在不同环境配置不同帐号地址等等，拿数据库举例datasource-dev.yml

```java
spring:
  datasource:
    name: datasource
    druid:
      url: jdbc:mysql://localhost:3306/ms_db?useUnicode=true&useSSL=false&characterEncoding=UTF-8&serverTimezone=UTC&useAffectedRows=true&allowPublicKeyRetrieval=true # 返回受影响记录条数
      username: root
      password: 1qaz@WSX
      driver-class-name: com.mysql.cj.jdbc.Driver
      initial-size: 1
      min-idle: 1
      max-active: 20
      max-wait: 60000
      time-between-eviction-runs-millis: 60000
      min-evictable-idle-time-millis: 300000
      validation-query: SELECT 'x'
      test-while-idle: true
      test-on-borrow: false
      test-on-return: false
      pool-prepared-statements: false
      max-pool-prepared-statement-per-connection-size: 20
      web-stat-filter:
        enabled: false
      stat-view-servlet:
        enabled: false
```

### 其他

springcloud的组件也会有一些公共的配置，例如ribbon、histrix等信息，这些配置都比较简单，可以考虑放在配置中心，也可以就放在项目中。