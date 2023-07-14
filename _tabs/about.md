---
title: 关于
dynamic_title: 毕成飞
icon: fas fa-info
order: 4
---

19 届 ｜ 湖北文理学院 ｜ 统招本科 ｜ 计算机科学与技术 ｜ 4 年经验 ｜ 1428976670@qq.com

求职城市：北京  
求职意向：JAVA  / 基础架构 / 全栈

## 技能

主要从事 JAVA 服务器端开发，大数据和前端也有过开发经验

1. 熟练使用 JAVA 基础，熟悉多线程、集合、JVM 等；
2. 熟悉常用框架，Spring Boot、Mybatis、JPA 等；
3. 熟悉 MySQL、Redis，熟悉 Shell；
4. 有大型微服务项目的开发、模块架构经验，熟练使用常见的缓存、消息等分布式技术；
5. 熟悉计算机基础，熟悉基本的数据结构、网络、算法；
6. 了解并使用过 Hadoop、HDFS、Spark、Hive 等；
7. 有前端开发能力；
8. 爱思考，实践性强，善于解决工作中的问题，可单独负责模块架构和开发

## 个人项目

- [https://github.com/bichengfei/EnumHandler](https://github.com/bichengfei/EnumHandler)
  
  MyBatis 中枚举类型处理器的扩展

## 工作经验

<B>2021/1 - 至今 &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;NewBanker &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; JAVA 开发</B>

财富管理平台开发、维护

综合营销运营平台中单独模块的设计、架构、开发，标准版私有化部署交付

<B>2019/6 - 2020/12 &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;   中科软 / 鹍骐 &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;    JAVA 开发</B>

JAVA 后台研发、部分前端开发、数据库设计

## 项目经验

##### 综合运营平台  &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;2022/1 - 至今

```
产品概述：针对基金公司的渠道营销获客平台
技术栈：JAVA、Spring Boot、Spring JPA、MySQL、Redis、xxl-job
主要工作内容：
  1. 接入客户的用户中心，进行单点登录；
  2. 设计、开发渠道用户黑名单；
  3. 设计、开发知识社区，内容、评论、点赞、收藏、关注、匿名、自定义作者；
  4. 设计、开发知识社区消息通知；
  5. 设计、开发知识社区积分排行榜；
  6. 开发直播管理；
  7. 负责标准版客户交付，主要进行客户数据对接；
难点：
  1. 知识库消息，评论时的消息通知，逻辑比较复杂，最后通过栈结构，顺利处理；
  2. 知识库积分排行榜，难点主要在于灵活配置、接口响应时间、实时更新，最终通过数据懒加载 + Redis 的 zset、String 、Hash 结构 + 数据埋点，做到实时响应
  3. 对系统中渠道、渠道联系人、部门、员工、基金产品、基金经理、短信的对接模块，重新做了架构。使用了：工厂模式 + 模版方法 + 代理模式 + 数据库事务 + Redis 锁 + xxl-job 主任务和子任务 + StopWatch + 同步任务的唯一标识
```

##### 财富管理平台 &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; 2021/1 - 至今

```
产品概述：针对财富管理公司的管理平台，SAAS + 私有化部署，覆盖 投资人小程序、投资人APP、理财师APP、WEB 管理后台、内容中心
技术栈：JAVA + Spring Boot + Mybatis + Dubbo + Zookeeper + Redis + MySQL
主要工作内容：
  1. 开发产品预约中心；
  2. 开发客户中心；
  3. 开发产品中心；
  4. 设计、开发积分商场；
  5. 私有化部署二次开发；
  6. SAAS 系统维护；
难点：
  1. 和另一位同事一起，维护一个近 50 个微服务，迭代近 5 年的 SAAS 系统，目前仍在正常运行，同时单人还需要维护 4 个大客户的私有化部署系统，提升了自身的代码阅读能力；
  2. B 端的产品预约单列表和导出优化，因为预约中心处理太过复杂，并且旧代码有坑，循环套循环，导致数据库连接全部被占用，其他业务被阻塞，最后通过 SQL 代替部分程序和 SQL 优化，提升查询效率和避免系统故障；
  3. 自定义 Mybatis 枚举类型处理器，提升工作效率；
```

##### 中国科技馆手机应用服务中心 &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; 2020/1 - 2020/8

```
项目概述：中国科技馆的移动应用平台
技术栈：JAVA + Spring Boot + Mybatis + MySQL + Hive + Hbase
主要工作内容：
  1. 开发 JAVA 接口；
  2. 开发 Hive 任务脚本；
难点：
  1. 对 Hive 函数不熟悉，任务脚本编写困难；
  2. Hive 计算有时出现阻塞，研究后发现是因为没有很好对使用任务队列功能
```

##### 广电统一运维管理平台 &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; 2019/6 - 2020/1

```
项目概述：对广电全国的物理设备进行性能监控，可视化大屏展示
技术栈：JAVA + Scala + Redis + WebSocket + Hadoop + Hdfs + Spark + Kafka + ElasticSearch
主要工作内容：
  1. 开发 JAVA 接口；
  2. 数据存储、数据分析、数据推送；
难点：
  1. 生产 Redis 集群出现单个节点带宽占满情况，研究后发现是因为 Redis 集群为哨兵模式，存在该缺陷
  2. 正式工作后对第一个项目，工程应用、团队协作、新技术的吸收，都很具有挑战性
```