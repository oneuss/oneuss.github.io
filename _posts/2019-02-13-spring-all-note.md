---
layout: post
title:  "玩转Spring全家桶 -- 学习笔记"
subtitle: "极客时间课程"
date:   2019-02-13
background: '/img/imac_bg.png'
---
![优惠](https://upload-images.jianshu.io/upload_images/13572633-440496cbd534cb5b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

#Spring家族
- Spring Framework

- Spring Boot
  快速构建基于Spring的应用程序
- Spring Cloud
  简化分布式系统的开发
#数据源
- HikariCP
  1. 特点
  高性能数据源，速度非常快
  2. HikariCP为什么这么快？
    - 字节码级别优化（JavaAssist生成）
    - 大量小改进
  3. https://github.com/brettwooldridge/HikariCP
- Alibaba Druid
  特点：  详细的监控，SQL防注入，内置加密配置，众多扩展点
#Spring 事务
##Spring 的事务抽象
1. 一致的事务抽象
- JDBC/Hibernate/MyBatis
- DataSource/JTA
2. 事务抽象的核心接口

    PlatformTransactionManager
    - DataSourceTransactionManager
    - HibernateTransactionManager
    - JtaTransactionManager

    TransactionDefinition
    - Propagation
    - Isolation
    - Timeout
    - Read-only status
```
void commit(TransactionStatus status) throws TransactionException;
void rollback(TransactionStatus status) throws TransactionException;
TransactionStatus getTransaction(@Nullable TransactionDefinition definition)  throws TransactionException;
```
3. 事务的传播特性

| 传播性 | 值 | 描述 |
| :------:| :------: | :------: |
| PROPAGATION_REQUIRED | 0 | 当前有事务就用当前的，没有就用新的 |
| PROPAGATION_SUPPORTS | 1 | 事务可有可无，不是必须的 |
| PROPAGATION_MANDATORY | 2 | 一定要有事务，不然就抛异常 |
| PROPAGATION_REQUIRES_NEW | 3 | 无论是否有事务，都起个新的事务（创建一个新的事务，并暂停当前事务（如果存在）） |
| PROPAGATION_NOT_SUPPORTED | 4 | 不支持事务，按非事务方式进行 |
| PROPAGATION_NEVER | 5 | 不支持事务，如果有事务则抛异常 |
| PROPAGATION_NESTED | 6 | 当前有事务就在当前事务里再起一个事务 |

4. 事务隔离级别

| 隔离性 |值 | 脏读 | 不可重复读 | 幻读 |
| :------:| :------: | :------: |:------: |:------: |
| ISOLATION_READ_UNCOMMITTED | 1 | 是 | 是 | × |
| ISOLATION_READ_COMMITTED | 2 | × | 是 | 是 |
| ISOLATION_REPEATABLE_READ | 3 | × | × | 是 |
| ISOLATION_SERIALIZABLE | 4 | × | × | × |

# ORM框架实践
1. Spring Data JPA
    Hibernate是JPA的一种实现。Spring Data JPA用的就是hibernate。
2. Lombok使用

## MyBatis 相关的一些工具
 1. MyBatis Generator
- 运行方式：
    - 命令行
java -jar mybatis-generator-core-x.x.x.jar -configfile generatorConfig.xml
    - Maven Plugin(mybatis-generator-maven-plugin)
mvn mybatis-generator:generator
${basedir}/src/main/resource/generatorConfig.xml
    - Eclipse Plugin
    - Java 程序
    - Ant Task  
2. pagehelper
https://github.com/pagehelper/Mybatis-PageHelper

# NoSQL 实践
## Docker辅助开发
