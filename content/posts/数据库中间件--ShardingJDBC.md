---
title: "数据库中间件--ShardingJDBC(附小demo)"
date: 2022-03-27T23:02:09+08:00
author: 周子胜
authorLink: https://github.com/zzs52
tags: ["技术分享","数据库中间件"]
categories: ["数据库中间件"]
draft: false
---

### 一、ShardingJDBC概述

官网：<http://shardingsphere.apache.org/index_zh.html>

#### 1.概述

Apache ShardingSphere 是一套开源的分布式数据库解决方案组成的生态圈，它由 JDBC、Proxy 和 Sidecar这 3 款既能够独立部署，又支持混合部署配合使用的产品组成。 它们均提供标准化的数据水平扩展、分布式事务和分布式治理等功能，可适用于如 Java 同构、异构语言、云原生等各种多样化的应用场景。

Apache ShardingSphere 5.x 版本开始致力于可插拔架构，项目的功能组件能够灵活地以可插拔的方式进行扩展。 目前，数据分片、读写分离、数据加密、影子库压测等功能，以及 MySQL、Oracle、PostgreSQL、SQLServer 等 SQL 与协议的支持，均通过插件的方式织入项目。 开发者能够像使用积木一样定制属于自己的独特系统。

ShardingSphere 已于2020年4月16日成为 Apache 软件基金会的顶级项目。

#### 2.认识ShardingJDBC

定位为轻量级 Java 框架，在 Java 的 JDBC 层提供的额外服务。 它使用客户端直连数据库，以 jar 包形式提供服务，无需额外部署和依赖，可理解为增强版的 JDBC 驱动，完全兼容基于 JDBC 的各种 ORM 框架，如：JPA，Mybatis，Hibernate，Spring JDBC Template 或直接使用 JDBC。

支持任何第三方的数据库连接池，如：DBCP,C3P0,Druid,HikariCP,BoneCP等。支持任意实现 JDBC 规范的数据库，目前支持 MySQL，Oracle，SQLServer，PostgreSQL 以及任何遵循 SQL92 标准的数据库。

![](E:\NoDishwashingStudio\fab03b882c0bcf69b83418b621213b70.png)

#### 3.ShardingJDBC功能架构图

![](E:\NoDishwashingStudio\18d3a43a4304b380da2dbd100f2eb7de.png)

#### 4.ShardingJDBC混合架构

![](E:\NoDishwashingStudio\8f40f91304567728fe19148a976cb4e6.png)

ShardingJDBC采用无中心化架构，适用于 Java 开发的高性能的轻量级OLTP（连接事务处理） 应用；ShardingProxy提供静态入口以及异构语言的支持，适用于OLAP（连接数据分析） 应用以及对分片数据库进行管理和运维的场景。

Apache ShardingSphere是多接入端共同组成的生态圈。通过混合使用 ShardingJDBC 和 ShardingProxy，并采用同一注册中心统一配置分片策略，能够灵活的搭建适用于各种场景的应用系统，使得架构师更加自由地调整适合于当前业务的最佳系统架构。

#### 5.ShardingSphere的功能清单

- 功能列表
  - 数据分片
  - 分库 & 分表
  - 读写分离
  - 分片策略定制化
  - 无中心化分布式主键
- 分布式事务
  - 标准化事务接口
  - XA强一致事务
  - 柔性事务
  - 数据库治理
- 分布式治理
  - 弹性伸缩
  - 可视化链路追踪
  - 数据加密

#### 6.ShardingSphere数据分片内核剖析

![](E:\NoDishwashingStudio\eeb4c4ddc2aa41e0e03360b3ddb5d239.png)

> SQL 解析

分为词法解析和语法解析。先通过词法解析器将 SQL 拆分为一个个不可再分的单词。再使用语法解析器对 SQL 进行理解，并最终解析出上下文。 解析上下文包括表、选择项、排序项、分组项、聚合函数、分页信息、查询条件以及可能需要修改的占位符的标记。

> 查询优化

合并和优化分片条件，如 OR 等。

> SQL路由

根据解析上下文匹配用户配置的分片策略，并生成路由路径。目前支持分片路由和广播路由。

> SQL改写

将 SQL 改写为在真实数据库中可以正确执行的语句。SQL 改写分为正确性改写和优化改写。

> SQL执行

通过多线程执行器异步执行。

> 结果归并

将多个执行结果集归并以便于通过统一的 JDBC 接口输出。结果归并包括流式归并、内存归并和使用装饰者模式的追加归并这几种方式。

### 二、ShardingJDBC准备 - Linux(centos7)安装MySQL5.7

在安装之前最好先更新一下源

```bash
yum update
```

可以选择在此路径下安装

```bash
cd /user/local/mysql
```

下载 安装用的MySQL官方的Yum Repository

```bash
wget -i -c http://dev.mysql.com/get/mysql57-community-release-el7-10.noarch.rpm
```

然后就可以直接安装了

```bash
yum -y install mysql57-community-release-el7-10.noarch.rpm
```

如果是新安装的MySQL，那么在开始安装MySQL服务器之前需要更新其GPG

```bash
rpm --import https://repo.mysql.com/RPM-GPG-KEY-mysql-2022
```

开始安装MySQL服务器

```bash
yum -y install mysql-community-server
```

安装成功后便可以进行MySQL数据库的设置

首先启动MySQL服务

```bash
systemctl start mysqld.service
```

查看MySQL运行状态

```bash
systemctl status mysqld.service
```

运行状态如图即可：

![](E:\NoDishwashingStudio\1e4aee34274d1c479f99f1473b4f2c5e.png)

使用MySQL初始密码登录数据库

```bash
grep "password" /var/log/mysqld.log

mysql -uroot -p(初始密码)

或者使用一步到位的命令

mysql -uroot -p$(awk '/temporary password/{print $NF}' /var/log/mysqld.log)
```

修改数据库密码

数据库默认密码规则必须携带大小写字母、特殊符号，字符长度大于8否则会报错。
因此设定较为简单的密码时需要首先修改set global validate_password_policy和_length参数值。

```bash
set global validate_password_policy=0;

set global validate_password_length=1;
```

修改密码

```bash
set password for root@localhost = password('新密码');

或者

ALTER USER 'root'@'localhost' IDENTIFIED BY '新密码';
```

退出后再登录进行测试

```bash
mysql -uroot -p(新密码)

show databases;
```

用可视化的客户端进行连接

先对数据库进行授权

```bash
GRANT ALL PRIVILEGES ON *.* TO 'root'@'%' IDENTIFIED BY '123456' WITH GRANT OPTION;

FLUSH PRIVILEGES;
```

启用FirewallD防火墙，开放3306端口，再重启

```bash
systemctl start firewalld.service

firewall-cmd --zone=public --add-port=3306/tcp --permanent

systemctl restart firewalld.service
```

到此就全部OK了(记得把云服务器安全组3306端口打开)

### 三、ShardingJDBC准备 - MySql完成主从复制

#### 1.概述

主从复制将一个MySQL数据库（主服务器）的数据复制到一个或多个MySQL数据库（从服务器）。

复制是异步的，从站不需要永久连接以接收来自主站的更新。

根据配置，可以复制主服务器中的所有数据库，所选的数据库，所选的表。

#### 2.主从复制的好处

- 横向扩展：在多个从站之间分配负载以提高性能。在此环境中，所有写入和更新都必须在主服务器上进行。但是，读取可以在一个或多个从服务器上进行。该架构可以提高写入性能（因为主服务器专注于更新），同时显著提高了从服务器的读取速度。
- 数据安全性：因为数据被复制到从站，并且从站可以暂停复制过程，所以可以在从站上运行备份服务而不会破坏相应的主数据。
- 分析：可以在主服务器上创建实时数据，而信息分析可以在从服务器上进行，而不会影响主服务器的性能。

#### 3.Replication的原理

![](E:\NoDishwashingStudio\50716df6ae118b97f18827b333c4ccda.png)

主服务器上面的任何修改都会通过自己的 I/O tread(I/O 线程)保存在二进制日志 Binary log 里面。

从服务器上面也启动一个 I/O thread，通过配置好的用户名和密码, 连接到主服务器上请求读取二进制日志，然后把读取到的二进制日志写到本地的一个Relay log（中继日志）里面。

从服务器上面同时开启一个 SQL thread 定时检查 Relay log(这个文件也是二进制的)，如果发现有更新立即把更新的内容在本机的数据库上面执行一遍。

从服务器设备负责决定应该执行二进制日志中的哪些语句。除非另行指定，否则主二进制日志中的所有事件都在从站上执行。如果需要，您可以将从服务器配置为仅处理一些特定数据库或表的事件。

#### 4.具体配置主从节点

master节点配置(master节点执行)

```bash
vim /etc/my.cnf
```

在该文件里添加如下内容

```cnf
[mysqld]
## 同一局域网内注意要唯一
server-id=100  
## 开启二进制日志功能，可以随便取（关键）
log-bin=mysql-bin
## 复制过滤：不需要备份的数据库，不输出（mysql库一般不同步）
binlog-ignore-db=mysql
## 为每个session 分配的内存，在事务过程中用来存储二进制日志的缓存
binlog_cache_size=1M
## 主从复制的格式（mixed,statement,row，默认格式是statement）
binlog_format=mixed
```

修改完该文件后要重启mysql服务

slave节点配置(slave节点执行)

```bash
vim /etc/my.cnf
```

在该文件里添加如下内容

```cnf
[mysqld]
## 设置server_id,注意要唯一
server-id=102
## 开启二进制日志功能，以备Slave作为其它Slave的Master时使用
log-bin=mysql-slave-bin
## relay_log配置中继日志
relay_log=edu-mysql-relay-bin
##复制过滤：不需要备份的数据库，不输出（mysql库一般不同步）
binlog-ignore-db=mysql
## 如果需要同步函数或者存储过程
log_bin_trust_function_creators=true
## 为每个session 分配的内存，在事务过程中用来存储二进制日志的缓存
binlog_cache_size=1M
## 主从复制的格式（mixed,statement,row，默认格式是statement）
binlog_format=mixed
## 跳过主从复制中遇到的所有错误或指定类型的错误，避免slave端复制中断。
## 如：1062错误是指一些主键重复，1032错误是因为主从数据库数据不一致
slave_skip_errors=1062
```

修改完该文件后要重启mysql服务

在master服务器授权slave服务器可以同步的权限(master节点执行)

进入mysql后执行

```bash
mysql> grant replication slave, replication client on *.* to 'root'@'slave服务的ip' identified by 'slave服务器的密码';

mysql> flush privileges;

# 查看MySQL现在有哪些用户及对应的IP权限
mysql> select user,host from mysql.user;
```

查询master服务器的binlog文件名和位置(master节点执行)

```bash
mysql> show master status;
```

![](E:\NoDishwashingStudio\94a6664c9053f2f16702a1e9d83c8da9.png)

日志文件名：mysql-bin.000002

复制的位置：2079

slave关联master节点(slave节点执行)

进入到slave节点

```bash
mysql -uroot -p你slave密码
```

开始绑定

```bash
mysql> change master to master_host='master服务器ip', master_user='root', master_password='master密码', master_port=3306, master_log_file='mysql-bin.000002',master_log_pos=2079;
```

启动主从复制(slave节点执行)

```bash
mysql> start slave;
```

再查看主从同步状态(slave节点执行)

```bash
mysql> show slave status\G;
```

![](E:\NoDishwashingStudio\5b2419b1d7977056090b2a4da7aeba4c.png)

### 四、ShardingJDBC的配置和读写分离实践

#### 1.内容大纲

- 新建一个springboot工程
- 修改application.properties为application.yml并配置application.yml
- 引入shardingJDBC相关的依赖、数据库驱动依赖等
- 定义entity、mapper、controller等
- 访问测试查看效果

#### 2.具体实现步骤

> 新建一个springboot工程

> 修改application.properties为application.yml并配置application.yml

```yaml
server:
  port: 8085
spring:
  main:
    allow-bean-definition-overriding: true
  shardingsphere:
    # 参数配置，显示sql
    props:
      sql:
        show: true
    # 配置数据源
    datasource:
      # 给每个数据源取别名，下面的ds0,ds1任意取名字
      names: ds0,ds1
      # 给master-ds0数据源配置数据库连接信息
      ds0:
        # 配置druid数据源
        type: com.alibaba.druid.pool.DruidDataSource
        driver-class-name: com.mysql.cj.jdbc.Driver
        url: jdbc:mysql://121.40.184.179:3306/zzs_order_db?useUnicode=true&characterEncoding=utf8&tinyInt1isBit=false&useSSL=false&serverTimezone=GMT
        username: root
        password: *
        maxPoolSize: 100
        minPoolSize: 5
      # 配置ds1-slave
      ds1:
        type: com.alibaba.druid.pool.DruidDataSource
        driver-class-name: com.mysql.cj.jdbc.Driver
        url: jdbc:mysql://121.40.100.114:3306/zzs_order_db?useUnicode=true&characterEncoding=utf8&tinyInt1isBit=false&useSSL=false&serverTimezone=GMT
        username: root
        password: *
        maxPoolSize: 100
        minPoolSize: 5
    # 配置默认数据源ds0
    sharding:
      # 默认数据源，主要用于写，注意一定要配置读写分离,注意：如果不配置，那么就会把这两个节点都当做从slave节点，新增，修改和删除会出错。
      default-data-source-name: ds0
#     配置数据源的读写分离，但是数据库一定要做主从复制
#    masterslave:
#      # 配置主从名称，可以任意取名字
#      name: ms
#      # 配置主库master，负责数据的写入
#      master-data-source-name: ds0
#      # 配置从库slave节点
#      slave-data-source-names: ds1
#      # 配置slave节点的负载均衡均衡策略，采用轮询机制(如果采用随机分配机制就用random_robin)
#      load-balance-algorithm-type: round_robin
# 整合mybatis的配置XXXXX
mybatis:
  mapper-locations: classpath:mapper/*.xml
  type-aliases-package: com.zzs.shardingJdbc.entity
```

> 引入相关依赖

```yaml
<properties>
    <java.version>1.8</java.version>
    <sharding-sphere.version>4.0.0-RC1</sharding-sphere.version>
</properties>
<!-- 依赖web -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
<!-- 依赖mybatis和mysql驱动 -->
<dependency>
    <groupId>org.mybatis.spring.boot</groupId>
    <artifactId>mybatis-spring-boot-starter</artifactId>
    <version>2.1.4</version>
</dependency>
<dependency>
    <groupId>mysql</groupId>
    <artifactId>mysql-connector-java</artifactId>
    <scope>runtime</scope>
</dependency>
<!--依赖lombok-->
<dependency>
    <groupId>org.projectlombok</groupId>
    <artifactId>lombok</artifactId>
    <optional>true</optional>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-test</artifactId>
    <scope>test</scope>
</dependency>
<!--依赖sharding-->
<dependency>
    <groupId>org.apache.shardingsphere</groupId>
    <artifactId>sharding-jdbc-spring-boot-starter</artifactId>
    <version>${sharding-sphere.version}</version>
</dependency>
<dependency>
    <groupId>org.apache.shardingsphere</groupId>
    <artifactId>sharding-core-common</artifactId>
    <version>${sharding-sphere.version}</version>
</dependency>
<!--依赖数据源druid-->
<dependency>
    <groupId>com.alibaba</groupId>
    <artifactId>druid-spring-boot-starter</artifactId>
    <version>1.1.21</version>
</dependency>
```

> 定义entity、mapper、controller等

entity

```java
package com.zzs.shardingJdbc.entity;

import lombok.Data;

import java.util.Date;

@Data
public class User {
    // 主键
    private Long id;
    // 昵称
    private String nickname;
    // 密码
    private String password;
    // 年龄
    private Integer age;
    // 性别
    private Integer sex;
    // 生日
    private Date birthday;
}
```

mapper

```java
package com.zzs.shardingJdbc.mapper;

import com.zzs.shardingJdbc.entity.User;
import org.apache.ibatis.annotations.Insert;
import org.apache.ibatis.annotations.Mapper;
import org.apache.ibatis.annotations.Options;
import org.apache.ibatis.annotations.Select;
import org.springframework.stereotype.Repository;

import java.util.List;

@Mapper
@Repository
public interface UserMapper {

    @Insert("insert into zzs_user(nickname,password,age,sex,birthday) values(#{nickname},#{password},#{age},#{sex},#{birthday})")
    @Options(useGeneratedKeys = true,keyColumn = "id",keyProperty = "id")
    void addUser(User user);

    @Select("select * from zzs_user")
    List<User> findUsers();
}
```

controller

```java
package com.zzs.shardingJdbc.controller;

import com.zzs.shardingJdbc.entity.User;
import com.zzs.shardingJdbc.mapper.UserMapper;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

import java.util.Date;
import java.util.List;
import java.util.Random;

@RestController
@RequestMapping("/user")
public class UserController {

    @Autowired
    private UserMapper userMapper;

    @GetMapping("/save")
    public String insert() {
        User user = new User();
        user.setNickname("zhangsan" + new Random().nextInt());
        user.setPassword("111111");
        user.setAge(4);
        user.setSex(2);
        user.setBirthday(new Date());
        userMapper.addUser(user);
        return "success";
    }

    @GetMapping("/listuser")
    public List<User> listuser() {
        return userMapper.findUsers();
    }
}
```

> 查看测试结果

访问 `http://localhost:8085/user/save` 一直进入到ds0主节点

访问 `http://localhost:8085/user/listuser` 一直进入到ds1节点

> run控制台查看日志

### 五、MySQL分库分表原理

#### 1.为什么要分库分表

解决高并发和数据量大的问题。

- 高并发情况下，会造成IO读写频繁，自然就会造成读写缓慢，甚至是宕机。一般单库不要超过2k并发，性能高的机器除外。
- 数据量大的问题，主要由于底层索引实现导致，MySQL的索引实现为B+Tree，数据量大会导致索引树十分庞大，造成查询缓慢。innodb的最大存储限制64TB。

#### 2.分库分表概念

分为垂直拆分和水平拆分。

**水平拆分：**同一个表的数据拆到不同的库不同的表中。可以根据时间、地区、或某个业务建维度，也可以通过hash进行拆分，最后通过路由访问到具体的数据。拆分后的每个表结构保持一致。

**垂直拆分：**就是把一个有很多字段的表给拆分成多个表，或者多个库上去。每个库表的结构都不一样，每个库表都包含部分字段。一般来说，可以根据业务维度进行拆分，如订单表可以拆分为订单、订单支持、订单地址、订单商品、订单扩展等表；也可以根据数据冷热程度拆分，20%的热点字段拆到一个表，80%的冷字段拆到另外一个表。

![](E:\NoDishwashingStudio\7f0d86e3bedbf0f6bf87d33056b9b93e.png)

#### 3.不停机分库分表数据迁移

一般数据库的拆分也是有一个过程的，一开始是单表，后面慢慢拆成多表。那么我们就看下如何平滑地从MySQL单表过度到MySQL的分库分表架构。

- 利用mysql+canal做增量数据同步，利用分库分表中间件，将数据路由到对应的新表中。

- 利用分库分表中间件，全量数据导入到对应的新表中。

- 通过单表数据和分库分表数据两两比较，更新不匹配的数据到新表中。

- 数据稳定后，将单表的配置切换到分库分表配置上。

![](E:\NoDishwashingStudio\74d7579fbb4fee984ef54d20d6bdf6e3.png)

#### 4.小结

垂直拆分：业务模块拆分，商品库，用户库，订单库等

水平拆分：对表进行水平拆分

表进行垂直拆分：表的字段过多，字段使用的频率不一（可以拆分两个表建立1 : 1关系）

### 六、ShardingJDBC的分库和分表实践

#### 1.逻辑表

逻辑表：水平拆分时用户数据根据用户id%2拆分为2个表，分别是：zzs_user0和zzs_user1。他们的逻辑表名是：zzs_user。

```yaml
    # 配置默认数据源ds0
    sharding:
      # 默认数据源，主要用于写，注意一定要配置读写分离 ,注意：如果不配置，那么就会把这两个节点都当做从slave节点，新增，修改和删除会出错。
      default-data-source-name: ds0
      # 配置分表的规则
      tables:
        # zzs_user 逻辑表名
        zzs_user:
```

#### 2.数据节点 - actual-data-nodes

```yaml
# 配置默认数据源ds0
    sharding:
      # 默认数据源，主要用于写，注意一定要配置读写分离 ,注意：如果不配置，那么就会把这两个节点都当做从slave节点，新增，修改和删除会出错。
      default-data-source-name: ds0
      # 配置分表的规则
      tables:
        # zzs_user 逻辑表名
        zzs_user:
          # 数据节点：数据源$->{0..N}.逻辑表名$->{0..N}
          actual-data-nodes: ds$->{0..1}.zzs_user$->{0..1}
```

数据节点是最小单元，由数据源名称和数据表组成，比如：ds0.zzs_user0

寻找某个数据的规则如下：

![](E:\NoDishwashingStudio\38bf8afce306037fed0f725894872910.png)

#### 3.分库分表5种分片策略

数据分片分为两种：

- 数据源分片
- 表分片

这是两个不同维度的分片规则，但是它们用的分片策略和规则是一样的。它们由两部分构成：

- 分片键
- 分片算法

##### 第一种：none

对应NoneShardingStragey(不分片策略)，SQL会被发给所有节点去执行，这个规则没有子项目可以配置。

##### 第二种：inline行表达式分片策略(核心，必须要掌握)

```yaml
server:
  port: 8085
spring:
  main:
    allow-bean-definition-overriding: true
  shardingsphere:
    # 参数配置，显示sql
    props:
      sql:
        show: true
    # 配置数据源
    datasource:
      # 给每个数据源取别名，下面的ds0,ds1任意取名字
      names: ds0,ds1
      # 给master-ds0每个数据源配置数据库连接信息
      ds0:
        # 配置druid数据源
        type: com.alibaba.druid.pool.DruidDataSource
        driver-class-name: com.mysql.cj.jdbc.Driver
        url: jdbc:mysql://121.40.184.179:3306/zzs_order_db?useUnicode=true&characterEncoding=utf8&tinyInt1isBit=false&useSSL=false&serverTimezone=GMT
        username: root
        password: *
        maxPoolSize: 100
        minPoolSize: 5
      # 配置ds1-slave
      ds1:
        type: com.alibaba.druid.pool.DruidDataSource
        driver-class-name: com.mysql.cj.jdbc.Driver
        url: jdbc:mysql://121.40.100.114:3306/zzs_order_db?useUnicode=true&characterEncoding=utf8&tinyInt1isBit=false&useSSL=false&serverTimezone=GMT
        username: root
        password: *
        maxPoolSize: 100
        minPoolSize: 5
    # 配置默认数据源ds0
    sharding:
      # 默认数据源，主要用于写，注意一定要配置读写分离 ,注意：如果不配置，那么就会把这两个节点都当做从slave节点，新增，修改和删除会出错。
      default-data-source-name: ds0
      # 配置分表的规则
      tables:
        # zzs_user 逻辑表名
        zzs_user:
          # 数据节点：数据源$->{0..N}.逻辑表名$->{0..N}
          actual-data-nodes: ds$->{0..1}.zzs_user$->{0..1}
          # 拆分库策略，也就是什么样子的数据放入放到哪个数据库中。
          database-strategy:
            inline:
              sharding-column: sex    # 分片字段（分片键）
              algorithm-expression: ds$->{sex % 2} # 分片算法表达式
          # 拆分表策略，也就是什么样子的数据放入放到哪个数据表中。
          table-strategy:
            inline:
              sharding-column: age    # 分片字段（分片键）
              algorithm-expression: zzs_user$->{age % 2} # 分片算法表达式
#    # 配置数据源的读写分离，但是数据库一定要做主从复制
#    masterslave:
#      # 配置主从名称，可以任意取名字
#      name: ms
#      # 配置主库master，负责数据的写入
#      master-data-source-name: ds0
#      # 配置从库slave节点
#      slave-data-source-names: ds1
#      # 配置slave节点的负载均衡均衡策略，采用轮询机制
#      load-balance-algorithm-type: round_robin
# 整合mybatis的配置XXXXX
mybatis:
  mapper-locations: classpath:mapper/*.xml
  type-aliases-package: com.zzs.shardingJdbc.entity
```

- 在每个数据源下的zzs_order_db库中建zzs_user0和zzs_user1表。
- 数据源规则：性别为偶数的放入ds0库，奇数的放入ds1库。
- 数据表规则：年龄为偶数的放入zzs_user0表中，奇数的放入zzs_user1表中。

##### 第三种：standard标准分片策略

**标准分片策略(了解即可)**

- 对应StrandardShardingStrategy，提供对SQL语句中的=，in和between and的分片操作支持。

  但StrandardShardingStrategy只支持分片键，但它提供了PreciseShardingAlgorithm和RangeShardingAlgorithm两个分片算法。

- PreciseShardingAlgorithm是必选的，用于处理=和in的分片。

- RangeShardingAlgorithm是可选的，是用于处理between and分片，如果不配置和RangeShardingAlgorithm，SQL的between and将按照全库路由处理。

![](E:\NoDishwashingStudio\0b4f58f74bac6b5baf843683f3213d20.png)

**自定义标准分片策略---按照日期(需要extend标准分片策略类)**

> 配置文件

```yam
server:
  port: 8085
spring:
  main:
    allow-bean-definition-overriding: true
  shardingsphere:
    # 参数配置，显示sql
    props:
      sql:
        show: true
    # 配置数据源
    datasource:
      # 给每个数据源取别名，下面的ds0,ds1任意取名字
      names: ds0,ds1
      # 给master-ds0每个数据源配置数据库连接信息
      ds0:
        # 配置druid数据源
        type: com.alibaba.druid.pool.DruidDataSource
        driver-class-name: com.mysql.cj.jdbc.Driver
        url: jdbc:mysql://121.40.184.179:3306/zzs_order_db?useUnicode=true&characterEncoding=utf8&tinyInt1isBit=false&useSSL=false&serverTimezone=GMT
        username: root
        password: *
        maxPoolSize: 100
        minPoolSize: 5
      # 配置ds1-slave
      ds1:
        type: com.alibaba.druid.pool.DruidDataSource
        driver-class-name: com.mysql.cj.jdbc.Driver
        url: jdbc:mysql://121.40.100.114:3306/zzs_order_db?useUnicode=true&characterEncoding=utf8&tinyInt1isBit=false&useSSL=false&serverTimezone=GMT
        username: root
        password: *
        maxPoolSize: 100
        minPoolSize: 5
    # 配置默认数据源ds0
    sharding:
      # 默认数据源，主要用于写，注意一定要配置读写分离 ,注意：如果不配置，那么就会把这两个节点都当做从slave节点，新增，修改和删除会出错。
      default-data-source-name: ds0
      # 配置分表的规则
      tables:
        # zzs_user 逻辑表名
        zzs_user:
          # 数据节点：数据源$->{0..N}.逻辑表名$->{0..N}
          actual-data-nodes: ds$->{0..1}.zzs_user$->{0..1}
          # 拆分库策略，也就是什么样子的数据放入放到哪个数据库中。
          database-strategy:
            standard:
              shardingColumn: birthday    # 分片字段（分片键）
              preciseAlgorithmClassName: com.zzs.shardingJdbc.algorithm.BirthdayAlgorithm # 分片算法表达式
          # 拆分表策略，也就是什么样子的数据放入放到哪个数据表中。
          table-strategy:
            inline:
              sharding-column: age    # 分片字段（分片键）
              algorithm-expression: zzs_user$->{age % 2} # 分片算法表达式
#    # 配置数据源的读写分离，但是数据库一定要做主从复制
#    masterslave:
#      # 配置主从名称，可以任意取名字
#      name: ms
#      # 配置主库master，负责数据的写入
#      master-data-source-name: ds0
#      # 配置从库slave节点
#      slave-data-source-names: ds1
#      # 配置slave节点的负载均衡均衡策略，采用轮询机制
#      load-balance-algorithm-type: round_robin
# 整合mybatis的配置XXXXX
mybatis:
  mapper-locations: classpath:mapper/*.xml
  type-aliases-package: com.zzs.shardingJdbc.entity
```

> 自定义的算法类

```java
package com.zzs.shardingJdbc.algorithm;

import org.apache.shardingsphere.api.sharding.standard.PreciseShardingAlgorithm;
import org.apache.shardingsphere.api.sharding.standard.PreciseShardingValue;

import java.util.ArrayList;
import java.util.Calendar;
import java.util.Collection;
import java.util.Date;
import java.util.Iterator;
import java.util.List;

public class BirthdayAlgorithm implements PreciseShardingAlgorithm<Date> {

    List<Date> dateList = new ArrayList<>();
    {
        Calendar calendar1 = Calendar.getInstance();
        calendar1.set(2022, 1, 1, 0, 0, 0);
        Calendar calendar2 = Calendar.getInstance();
        calendar2.set(2023, 1, 1, 0, 0, 0);
        dateList.add(calendar1.getTime());
        dateList.add(calendar2.getTime());
    }

    @Override
    public String doSharding(Collection<String> collection, PreciseShardingValue<Date> preciseShardingValue) {
        // 获取属性数据库的值
        Date date = preciseShardingValue.getValue();
        // 获取数据源的名称信息列表
        Iterator<String> iterator = collection.iterator();
        String target = null;
        for (Date s : dateList) {
            target = iterator.next();
            // 如果数据晚于指定的日期直接返回
            if (date.before(s)) {
                break;
            }
        }
        return target;
    }
}
```

> 查看测试结果

##### 第四种：complex复合分片策略（了解）

- 对应ComplexShardingStrategy。复合分片策略提供对SQL语句中的=，in和between and的分片操作支持。
- ComplexShardingStrategy支持多分片键，由于多分片键之间的关系复杂，因此并未进行过多的封装，完全由开发者自己实现，提供最大的灵活度。

![](E:\NoDishwashingStudio\69613e757df2a6939e92087fde2097ba.png)

##### 第五种：hint分片策略（了解）

- 对应接口：HintShardingStrategy。通过Hint而非SQL解析的方式的分片策略。
- 对于分片字段非SQL决定，而是由其他外置条件决定的场景，可使用SQL hint灵活的注入分片字段。例如：按照用户登录的时间，主键等进行分库，而数据库中并无此字段。SQL hint支持通过Java API和SQL注解两种方式使用。让分库分表更加灵活。

![](E:\NoDishwashingStudio\bc9bda0c7c525216b91af7cbf38cae11.png)

### 七、分布式主键配置

ShardingSphere提供灵活地配置分布式主键生成策略方式，在分片规则配置模块可配置每个表的主键生成策略。默认使用雪花算法，（snowflake）生成64bit的长整型数据。支持两种方式配置：

- SNOWFLAKE
- UUID

这里切记：主键列不能自增长。数据类型是：bigint(20)

```yaml
# 配置默认数据源ds0
    sharding:
      # 默认数据源，主要用于写，注意一定要配置读写分离 ,注意：如果不配置，那么就会把这两个节点都当做从slave节点，新增，修改和删除会出错。
      default-data-source-name: ds0
      # 配置分表的规则
      tables:
        # zzs_user 逻辑表名
        zzs_user:
          key-generator:
            # 主键的列明，
            column: id
            type: SNOWFLAKE
          # 数据节点：数据源$->{0..N}.逻辑表名$->{0..N}
          actual-data-nodes: ds$->{0..1}.zzs_user$->{0..1}
```

> 执行查看结果

### 八、分库分表 - 按年月来分案例

#### 1.规则类

```java
package com.zzs.shardingJdbc.algorithm;

import org.apache.shardingsphere.api.sharding.standard.PreciseShardingAlgorithm;
import org.apache.shardingsphere.api.sharding.standard.PreciseShardingValue;

import java.util.Collection;

public class YearMonthShardingAlgorithm implements PreciseShardingAlgorithm<String> {

    private static final String SPLITTER = "_";

    @Override
    public String doSharding(Collection availableTargetNames, PreciseShardingValue shardingValue) {
        String tbName = shardingValue.getLogicTableName() + "_" + shardingValue.getValue();
        System.out.println("Sharding input:" + shardingValue.getValue() + ", output:{}" + tbName);
        return tbName;
    }
}
```

#### 2.entity

```java
package com.zzs.shardingJdbc.entity;

import lombok.Data;

import java.util.Date;

@Data
public class UserOrder {
    // 主键
    private Long orderid;
    // 用户ID
    private Long userid;
    // 订单编号
    private String ordernumber;
    // 创建时间
    private Date createTime;
    // 年月
    private String yearmonth;
}
```

#### 3.mapper

```java
package com.zzs.shardingJdbc.mapper;

import com.zzs.shardingJdbc.entity.UserOrder;
import org.apache.ibatis.annotations.Insert;
import org.apache.ibatis.annotations.Mapper;
import org.apache.ibatis.annotations.Options;
import org.springframework.stereotype.Repository;

@Mapper
@Repository
public interface UserOrderMapper {

    @Insert("insert into zzs_user_order(userid,ordernumber,createTime,yearmonth) values(#{userid},#{ordernumber},#{createTime},#{yearmonth})")
    @Options(useGeneratedKeys = true,keyColumn = "orderid",keyProperty = "orderid")
    void addUserOrder(UserOrder userOrder);
}
```

#### 4.配置

```yaml
server:
  port: 8085
spring:
  main:
    allow-bean-definition-overriding: true
  shardingsphere:
    # 参数配置，显示sql
    props:
      sql:
        show: true
    # 配置数据源
    datasource:
      # 给每个数据源取别名，下面的ds0,ds1任意取名字
      names: ds0,ds1
      # 给master-ds0每个数据源配置数据库连接信息
      ds0:
        # 配置druid数据源
        type: com.alibaba.druid.pool.DruidDataSource
        driver-class-name: com.mysql.cj.jdbc.Driver
        url: jdbc:mysql://121.40.184.179:3306/zzs_order_db?useUnicode=true&characterEncoding=utf8&tinyInt1isBit=false&useSSL=false&serverTimezone=GMT
        username: root
        password: *
        maxPoolSize: 100
        minPoolSize: 5
      # 配置ds1-slave
      ds1:
        type: com.alibaba.druid.pool.DruidDataSource
        driver-class-name: com.mysql.cj.jdbc.Driver
        url: jdbc:mysql://121.40.100.114:3306/zzs_order_db?useUnicode=true&characterEncoding=utf8&tinyInt1isBit=false&useSSL=false&serverTimezone=GMT
        username: root
        password: *
        maxPoolSize: 100
        minPoolSize: 5
    # 配置默认数据源ds0
    sharding:
      # 默认数据源，主要用于写，注意一定要配置读写分离 ,注意：如果不配置，那么就会把这两个节点都当做从slave节点，新增，修改和删除会出错。
      default-data-source-name: ds0
      # 配置分表的规则
      tables:
        zzs_user_order:
          # 数据节点：数据源$->{0..N}.逻辑表名$->{0..N}
          actual-data-nodes: ds0.zzs_user_order_$->{2021..2022}${(1..3).collect{t ->t.toString().padLeft(2,'0')} }
          key-generator:
            column: orderid
            type: SNOWFLAKE
          # 拆分表策略，也就是什么样子的数据放入放到哪个数据表中。
          table-strategy:
            standard:
              shardingColumn: yearmonth
              preciseAlgorithmClassName: com.zzs.shardingJdbc.algorithm.YearMonthShardingAlgorithm
mybatis:
  mapper-locations: classpath:mapper/*.xml
  type-aliases-package: com.zzs.shardingJdbc.entity
```

#### 5.测试

```java
package com.zzs.shardingJdbc;

import com.zzs.shardingJdbc.entity.UserOrder;
import com.zzs.shardingJdbc.mapper.UserOrderMapper;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;

import java.util.Date;
import java.util.Random;

@SpringBootTest
class ShardingJdbcApplicationTests {

    @Autowired
    private UserOrderMapper userOrderMapper;

    @Test
    public void orderyearMaster() {
        UserOrder userOrder = new UserOrder();
        userOrder.setUserid(1L);
        userOrder.setOrdernumber("133455678");
        userOrder.setCreateTime(new Date());
        userOrder.setYearmonth("202203");
        userOrderMapper.addUserOrder(userOrder);
    }
}
```

如果不使用标准分片策略(standard)，而选择使用inline分片策略，则不需要写规则类，再将配置文件修改为：

```yaml
server:
  port: 8085
spring:
  main:
    allow-bean-definition-overriding: true
  shardingsphere:
    # 参数配置，显示sql
    props:
      sql:
        show: true
    # 配置数据源
    datasource:
      # 给每个数据源取别名，下面的ds0,ds1任意取名字
      names: ds0,ds1
      # 给master-ds0每个数据源配置数据库连接信息
      ds0:
        # 配置druid数据源
        type: com.alibaba.druid.pool.DruidDataSource
        driver-class-name: com.mysql.cj.jdbc.Driver
        url: jdbc:mysql://121.40.184.179:3306/zzs_order_db?useUnicode=true&characterEncoding=utf8&tinyInt1isBit=false&useSSL=false&serverTimezone=GMT
        username: root
        password: *
        maxPoolSize: 100
        minPoolSize: 5
      # 配置ds1-slave
      ds1:
        type: com.alibaba.druid.pool.DruidDataSource
        driver-class-name: com.mysql.cj.jdbc.Driver
        url: jdbc:mysql://121.40.100.114:3306/zzs_order_db?useUnicode=true&characterEncoding=utf8&tinyInt1isBit=false&useSSL=false&serverTimezone=GMT
        username: root
        password: *
        maxPoolSize: 100
        minPoolSize: 5
    # 配置默认数据源ds0
    sharding:
      # 默认数据源，主要用于写，注意一定要配置读写分离 ,注意：如果不配置，那么就会把这两个节点都当做从slave节点，新增，修改和删除会出错。
      default-data-source-name: ds0
      # 配置分表的规则
      tables:
        zzs_user_order:
          # 数据节点：数据源$->{0..N}.逻辑表名$->{0..N}
          actual-data-nodes: ds0.zzs_user_order_$->{2021..2022}${(1..3).collect{t ->t.toString().padLeft(2,'0')} }
          key-generator:
            column: orderid
            type: SNOWFLAKE
          # 拆分表策略，也就是什么样子的数据放入放到哪个数据表中。
          table-strategy:
            inline:
              shardingColumn: yearmonth
              algorithmExpression: zzs_user_order_$->{yearmonth}
mybatis:
  mapper-locations: classpath:mapper/*.xml
  type-aliases-package: com.zzs.shardingJdbc.entity
```

### 九、ShardingJDBC的事务管理

#### 1.分布式事务的简单认识

数据库事务需要满足ACID（原子性、一致性、隔离性、持久性）四个特性。

- 原子性（Atomicity）指事务作为整体来执行，要么全部执行，要么全不执行。
- 一致性（Consistency）指事务应确保数据从一个一致的状态转变为另一个一致的状态。
- 隔离性（Isolation）指多个事务并发执行时，一个事务的执行不应影响其他事务的执行。
- 持久性（Durability）指已提交的事务修改数据会被持久保存。

在单一数据节点中，事务仅限于对单一数据库资源的访问控制，称之为本地事务。几乎所有的成熟的关系型数据库都提供了对本地事务的原生支持。 但是在基于微服务的分布式应用环境下，越来越多的应用场景要求对多个服务的访问及其相对应的多个数据库资源能纳入到同一个事务当中，分布式事务应运而生。关系型数据库虽然对本地事务提供了完美的ACID原生支持。 但在分布式的场景下，它却成为系统性能的桎梏。如何让数据库在分布式场景下满足ACID的特性或找寻相应的替代方案，是分布式事务的重点工作。

> 本地事务

在不开启任何分布式事务管理器的前提下，让每个数据节点各自管理自己的事务。 它们之间没有协调以及通信的能力，也并不互相知晓其他数据节点事务的成功与否。 本地事务在性能方面无任何损耗，但在强一致性以及最终一致性方面则力不从心。

> 两阶段提交

XA协议，最早的分布式事务模型是由X/Open国际联盟提出的X/Open Distributed Transaction Processing（DTP）模型，简称XA协议。

基于XA协议实现的分布式事务对业务侵入很小。 它最大的优势就是对使用方透明，用户可以像使用本地事务一样使用基于XA协议的分布式事务。 XA协议能够严格保障事务ACID特性。

严格保障事务ACID特性是一把双刃剑。 事务执行在过程中需要将所需资源全部锁定，它更加适用于执行时间确定的短事务。 对于长事务来说，整个事务进行期间对数据的独占，将导致对热点数据依赖的业务系统并发性能衰退明显。 因此，在高并发的性能至上场景中，基于XA协议的分布式事务并不是最佳选择。

> 柔性事务

如果将实现了ACID的事务要素的事务称为刚性事务的话，那么基于BASE事务要素的事务则称为柔性事务。 BASE是基本可用、柔性状态和最终一致性这三个要素的缩写。

基本可用（Basically Available）保证分布式事务参与方不一定同时在线。
柔性状态（Soft state）则允许系统状态更新有一定的延时，这个延时对客户来说不一定能够察觉。
而最终一致性（Eventually consistent）通常是通过消息传递的方式保证系统的最终一致性。

在ACID事务中对隔离性的要求很高，在事务执行过程中，必须将所有的资源锁定。 柔性事务的理念则是通过业务逻辑将互斥锁操作从资源层面上移至业务层面。通过放宽对强一致性要求，来换取系统吞吐量的提升。

基于ACID的强一致性事务和基于BASE的最终一致性事务都不是银弹，只有在最适合的场景中才能发挥它们的最大长处。 可通过下表详细对比它们之间的区别，以帮助开发者进行技术选型。

#### 2.具体案例

> 导入分布式事务的依赖

```pom
 <!--依赖sharding-->
 <dependency>
     <groupId>io.shardingsphere</groupId>
     <artifactId>sharding-transaction-spring-boot-starter</artifactId>
     <version>3.1.0</version>
 </dependency>
```

> 事务的几种类型

**本地事务**

- 完全支持非跨库事务，例如：仅分表或分库但是路由的结果在单库中。
- 完全支持因逻辑异常导致的跨库事务。例如：同一事务中，跨两个库更新，更新完毕后，抛出空指针，则两个库的内容都能回滚。
- 不支持因网络、硬件异常导致的跨库事务。例如：同一事务中，跨两个库更新，更新完毕后，未提交之前，第一个库宕机，则只有第二个库数据提交。

**两阶段XA事务**

- 支持数据分片后的跨库XA事务
- 两阶段提交保证操作的原子性和数据的强一致性
- 服务宕机重启后，提交/回滚中的事务可自动恢复
- SPI机制整合主流的XA事务管理器，默认Atomikos，可以选择使用Narayana和Bitronix
  同时支持XA和非XA的连接池
- 提供spring-boot和namespace的接入端

- 不支持服务宕机后，在其它机器上恢复提交/回滚中的数据

**Seata柔性事务**

- 完全支持跨库分布式事务
- 支持RC隔离级别
- 通过undo快照进行事务回滚
- 支持服务宕机后的，自动恢复提交中的事务

依赖：

- 需要额外部署Seata-server服务进行分支事务的协调
  待优化项
- ShardingSphere和Seata会对SQL进行重复解析

> Order实体类

```java
package com.zzs.shardingJdbc.entity;

import lombok.Data;

import java.util.Date;

@Data
public class Order {
    // 主键
    private Long orderid;
    // 订单编号
    private String ordernumber;
    // 用户ID
    private Long userid;
    // 产品id
    private Long productid;
    // 创建时间
    private Date createTime;
}
```

> OrderMapper

```java
package com.zzs.shardingJdbc.mapper;

import com.zzs.shardingJdbc.entity.Order;
import org.apache.ibatis.annotations.Insert;
import org.apache.ibatis.annotations.Mapper;
import org.apache.ibatis.annotations.Options;
import org.springframework.stereotype.Repository;

@Mapper
@Repository
public interface OrderMapper {

    @Insert("insert into zzs_order(ordernumber,userid,productid,createTime) values(#{ordernumber},#{userid},#{productid},#{createTime})")
    @Options(useGeneratedKeys = true,keyColumn = "orderid",keyProperty = "orderid")
    void addOrder(Order order);
}
```

> 配置文件

```yam
server:
  port: 8085
spring:
  main:
    allow-bean-definition-overriding: true
  shardingsphere:
    # 参数配置，显示sql
    props:
      sql:
        show: true
    # 配置数据源
    datasource:
      # 给每个数据源取别名，下面的ds0,ds1任意取名字
      names: ds0,ds1
      # 给master-ds0每个数据源配置数据库连接信息
      ds0:
        # 配置druid数据源
        type: com.alibaba.druid.pool.DruidDataSource
        driver-class-name: com.mysql.cj.jdbc.Driver
        url: jdbc:mysql://121.40.184.179:3306/zzs_order_db?useUnicode=true&characterEncoding=utf8&tinyInt1isBit=false&useSSL=false&serverTimezone=GMT
        username: root
        password: *
        maxPoolSize: 100
        minPoolSize: 5
      # 配置ds1-slave
      ds1:
        type: com.alibaba.druid.pool.DruidDataSource
        driver-class-name: com.mysql.cj.jdbc.Driver
        url: jdbc:mysql://121.40.100.114:3306/zzs_order_db?useUnicode=true&characterEncoding=utf8&tinyInt1isBit=false&useSSL=false&serverTimezone=GMT
        username: root
        password: *
        maxPoolSize: 100
        minPoolSize: 5
    # 配置默认数据源ds0
    sharding:
      # 默认数据源，主要用于写，注意一定要配置读写分离 ,注意：如果不配置，那么就会把这两个节点都当做从slave节点，新增，修改和删除会出错。
      default-data-source-name: ds0
      # 配置分表的规则
      tables:
        # zzs_user 逻辑表名
        zzs_user:
          key-generator:
            # 主键的列明，
            column: id
            type: SNOWFLAKE
          # 数据节点：数据源$->{0..N}.逻辑表名$->{0..N}
          actual-data-nodes: ds$->{0..1}.zzs_user$->{0..1}
          # 拆分库策略，也就是什么样子的数据放入放到哪个数据库中。
          database-strategy:
            standard:
              shardingColumn: birthday    # 分片字段（分片键）
              preciseAlgorithmClassName: com.zzs.shardingJdbc.algorithm.BirthdayAlgorithm # 分片算法表达式
          # 拆分表策略，也就是什么样子的数据放入放到哪个数据表中。
          table-strategy:
            inline:
              sharding-column: age    # 分片字段（分片键）
              algorithm-expression: zzs_user$->{age % 2} # 分片算法表达式
        zzs_order:
          # 数据节点：数据源$->{0..N}.逻辑表名$->{0..N}
          actual-data-nodes: ds0.zzs_order$->{0..1}
          key-generator:
            column: orderid
            type: SNOWFLAKE
          # 拆分表策略，也就是什么样子的数据放入放到哪个数据表中。
          table-strategy:
            inline:
              sharding-column: orderid    # 分片字段（分片键）
              algorithm-expression: zzs_order$->{orderid % 2} # 分片算法表达式
        zzs_user_order:
          # 数据节点：数据源$->{0..N}.逻辑表名$->{0..N}
          actual-data-nodes: ds0.zzs_user_order_$->{2021..2022}${(1..3).collect{t ->t.toString().padLeft(2,'0')} }
          key-generator:
            column: orderid
            type: SNOWFLAKE
          # 拆分表策略，也就是什么样子的数据放入放到哪个数据表中。
          table-strategy:
#            inline:
#              shardingColumn: yearmonth
#              algorithmExpression: zzs_user_order_$->{yearmonth}
            standard:
              shardingColumn: yearmonth
              preciseAlgorithmClassName: com.zzs.shardingJdbc.algorithm.YearMonthShardingAlgorithm
# 整合mybatis的配置XXXXX
mybatis:
  mapper-locations: classpath:mapper/*.xml
  type-aliases-package: com.zzs.shardingJdbc.entity
```

> service层代码编写

```java
package com.zzs.shardingJdbc.service;

import com.zzs.shardingJdbc.entity.Order;
import com.zzs.shardingJdbc.entity.User;
import com.zzs.shardingJdbc.mapper.OrderMapper;
import com.zzs.shardingJdbc.mapper.UserMapper;
import io.shardingsphere.transaction.annotation.ShardingTransactionType;
import io.shardingsphere.transaction.api.TransactionType;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

@Service
public class UserOrderService {

    @Autowired
    private UserMapper userMapper;

    @Autowired
    private OrderMapper orderMapper;

    @ShardingTransactionType(TransactionType.XA)
    @Transactional(rollbackFor = Exception.class)
    public int saveUserOrder(User user, Order order) {
        userMapper.addUser(user);
        order.setUserid(user.getId());
		// int a = 1/0; //测试回滚，统一提交的话，将这行注释掉就行
        orderMapper.addOrder(order);
        return 1;
    }
}
```

> 测试

```java
package com.zzs.shardingJdbc;

import com.zzs.shardingJdbc.entity.Order;
import com.zzs.shardingJdbc.entity.User;
import com.zzs.shardingJdbc.service.UserOrderService;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;

import java.util.Date;
import java.util.Random;

@SpringBootTest
class ShardingJdbcApplicationTests {

    @Autowired
    private UserOrderService userOrderService;

    @Test
    void contextLoads() throws Exception {
        User user = new User();
        user.setNickname("zhangsan" + new Random().nextInt());
        user.setPassword("1234567");
        user.setSex(1);
        user.setAge(2);
        user.setBirthday(new Date());
        Order order = new Order();
        order.setOrdernumber("133455678");
        order.setProductid(1234L);
        order.setCreateTime(new Date());
        userOrderService.saveUserOrder(user, order);
    }
}
```

还有一些相关面试题和知识点可参考文章：<https://blog.csdn.net/qq_44866424/article/details/120009099>

视频可参考狂神说ShardingJDBC讲解：<https://www.bilibili.com/video/BV1ei4y1K7dn>
