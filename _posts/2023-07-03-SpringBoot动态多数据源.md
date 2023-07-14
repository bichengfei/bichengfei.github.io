```yaml
title: Spring Boot 动态多数据源
author: BiChengfei
date: 2023-07-03 14:53:00 +0800
categories: [JAVA]
tags: [JAVA, Spring Boot, MySQL]
pin: true
```

[https://github.com/bichengfei/dynamic-ds](https://github.com/bichengfei/dynamic-ds)

1. 把租户唯一标识存储在`ThreadLocal`中，然后实现`javax.sql.DataSource`，重写`getConnection`方法，使得不同租户，获取不同的数据库连接

2. 在新增租户的时候，JPA 可以自动生成表结构
