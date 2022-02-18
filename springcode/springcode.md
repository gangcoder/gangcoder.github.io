<!-- ---
title: Spring Code
date: 2021-07-15 02:23:57
category: java100, springcode
--- -->

# Spring Code

## 内容

- IOC
  - [源码阅读环境](/springcode/spring_start.md)
  - [XML 应用上下文启动实现](/springcode/spring_xml_application_context.md)
  - [refresh 方法解析](/springcode/spring_refresh.md)
  - [BeanDefinition 解析实现](/springcode/load_bean_definition.md)
  - [PostProcessor 总结](/springcode/spring_post_processor.md)
  - [Bean 实例化过程](/springcode/spring_bean_init.md)
  - [Bean 属性与Aware 注入](/springcode/spring_bean_populate.md)
    - Aware 接口
  - [自动装配实现](/springcode/spring_autowire.md)
  - [占位符替换实现](/springcode/spring_placeholder.md)
  - 环境对象和监听器的深入讲解
  - [包扫描标签component-scan 解析](/springcode/spring_component_scan.md)
  - [AnnotationConfig 注解驱动上下文实现](/springcode/spring_annotation_config.md)
  - [@Configuration 注解解析](/springcode/spring_configurationclass.md)
- AOP
  - [Spring AOP 基于XML 启用原理](/springcode/spring_aop_xml.md)
  - [Spring AOP 代理实例化与调用](/springcode/spring_aop_invoke.md)
  - [Spring AOP 基于注解启用原理](/springcode/spring_aop_annotation.md)

## 记录

2021
- 11/22 Spring是如何加载配置文件到应用程序的
- 11/24 p8
- 11/25 p10

- 12/27 AOP 整理
- 12/28 IOC 阅读

2022
- 1/20 IOC 和AOP 整理绘图完成

## 参考

https://www.bilibili.com/video/BV1JQ4y1Z79h?p=2

https://www.bilibili.com/video/BV1K3411C7dn

https://www.bilibili.com/video/BV1Pb4y1k7FZ?p=2

- Spring IOC
- Spring AOP
  - Spring 源码深入解析（第2版）
  - AOP
  - AspectJ
  - Spring 揭秘

### 展现形式

1. 需要UML 类图，展示关键 Method 和Field 以及继承关系，从类关系中提取
2. 基于流程图展示调用逻辑，从代码中提取


