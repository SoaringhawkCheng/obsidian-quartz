---
publish: true
title: 「Spring框架体系」学习笔记
created: 2023-04-03 18:18:51
modified: 2026-11-04T20:22:59.377Z
---

# 一、Spring

# 二、Spring Boot

## 2.1 Jar包启动原理

### 2.1.1 目录结构

BOOT-INF/classes：目录存放应用编译后的class文件。

BOOT-INF/lib：目录存放应用依赖的第三方JAR包文件。

META-INF：目录存放应用打包信息(Maven坐标、pom文件)和MANIFEST.MF文件。

org：目录存放SpringBoot相关class文件。

![[软件开发/框架中间件/素材附件/Pasted image 20261105041824.png]]

### 2.1.2 清单文件 MANIFEST.MF

![[软件开发/框架中间件/素材附件/Pasted image 20261105041831.png]]

Main-Class是入口类

### 2.1.3 索引文件 idx

classpath.idx: jar包添加到classpath的顺序

![[软件开发/框架中间件/素材附件/Pasted image 20261105041837.png]]

layers.idx: 允许将 jar 拆分为逻辑层以创建Docker/OCI镜像

![[软件开发/框架中间件/素材附件/Pasted image 20261105041842.png]]

## 2.2 Spring Boot启动原理

[[「Spring Boot启动源码」学习笔记]]

# 三、Spring Cloud

## 3.1 Sentinel

[[「Spring Cloud Sentinel原理」学习笔记]]

# 四、其他一些零散稳定

[ApplicationContext容器源码解析：spring启动时是如何解析xml配置文件并注册bean的？](https://zhuanlan.zhihu.com/p/358803204)

[spring boot自动装配原理](https://zhuanlan.zhihu.com/p/503007698)

[Spring基础组件的使用——@ComponentScan扫描规则及其源码分析](https://blog.csdn.net/qq_27610647/article/details/115704426)

[超级重要的ConfigurationClassPostProcessor](https://blog.csdn.net/aqin1012/article/details/126053792)

todo 看完自动化配置源码
[深入Spring Boot ：从源码分析自动配置原理](https://zhuanlan.zhihu.com/p/559705017)
