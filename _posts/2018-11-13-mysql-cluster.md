---
layout: post
title: mysql集群方案
---



mysql集群有两种集群方式： innodb集群和ndb cluster，innodb方式相对简单一些，ndb更加符合分布式部署的原则

## innodb集群

### 概述

![架构](/images/innodb_cluster_overview.png)

### 原理

借助mysqlshell里的adminapi配置和管理多个服务器实例，充当innodb集群。

使用innodb集群的应用结构

![2](/images/mysql-router-positioning.png)

### 安装

#### 安装和升级mysql

安装mysqlserver

```shell
sudo yum install mysql-community-server
```

或者下载rpm包

```shell
curl https://dev.mysql.com/get/mysql80-community-release-el7-1.noarch.rpm
```

安装rpm包

```

```

启动mysql

```shell
sudo service mysqld start
```



#### 安装mysql shell

下载rpm包

```shell
curl https://dev.mysql.com/get/Downloads/MySQL-Shell/mysql-shell-8.0.13-1.el7.x86_64.rpm
```

安装rpm包

```
sudo rpm -ivh mysql-shell-8.0.13-1.el7.x86_64.rpm
```



#### 安装MysqlRouter

MySQLRouter是InnoDB集群的一部分，是轻量级中间件，可在应用程序和后端MySQL服务器之间提供透明路由。

YUM安装

- 下载rpm包

  ```
  curl https://dev.mysql.com/get/Downloads/MySQL-Router/mysql-router-community-8.0.13-1.el7.x86_64.rpm
  ```

  其他可选版本有2.1.6和2.0.4

- 安装rpm包

  ```
  sudo rpm -i mysql-router-community-8.0.13-1.el7.x86_64.rpm
  ```

### 配置

#### 配置mysql实例

在每个mysql机器上配置集群实例

```shell
dba.configureInstance([instance][, options])
```

```shell
mysql-js> dba.configureInstance('ic@ic-1:3306', \ 
{clusterAdmin: "'icadmin'@'ic-1'%", clusterAdminPassword: 'password'});
```

#### 创建mysql集群

在主节点上创建集群

```shell
var cluster = dba.createCluster('testCluster')
```

#### mysql实例加入集群

在主节点上把其他节点都加入到集群当中

```shell
cluster.addInstance('ic@ic-2:3306')
```

检查集群状态

```shell
cluster.status()
```

#### 设置mysqlrouter

在应用部署的机器上运行,配置连接朱节点的router

```shell
 mysqlrouter --bootstrap root@localhost:3310 --directory /tmp/myrouter
  cd /tmp/myrouter

./start.sh
```

#### springboot使用mysqlrouter

mysqlrouter有两个端口，一个用来写，一个用来读，读端口会对请求做负载均衡

在springboot中需要配置两个数据源，两个数据源只有端口号不同，在应用中需要根据操作数据库的类型（读/写）切换数据源来实现读写分离。

## mysql-ndb-cluster



