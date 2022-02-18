<!-- ---
title: 源码阅读环境
date: 2021-09-07 09:09:42
category: java100, springcode
--- -->

# 源码阅读环境

1. 安装 gradle 并更新依赖
2. 创建测试 module

## 测试代码

测试代码:

```java
// Hello.class
package com.hello;
public class Hello {
	public void sayHello() {
		System.out.println("Hello, spring");
	}
}


// MyApplication.class
package com.hello;

import org.springframework.context.ApplicationContext;
import org.springframework.context.support.ClassPathXmlApplicationContext;

public class MyApplication {
	public static void main(String[] args) {
		ApplicationContext ac = new ClassPathXmlApplicationContext("classpath:applicationContext.xml");
		Hello hello = (Hello)ac.getBean("hello");
		hello.sayHello();
	}
}
```

Bean 定义文件 applicationContext.xml:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
	   xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	   xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd">
	<bean id="hello" class="com.hello.Hello"></bean>
</beans>
```

build.gradle 依赖配置:

```
plugins {
    id 'java'
}

group 'org.springframework'
version '5.2.15.RELEASE'

repositories {
    mavenCentral()
}

dependencies {
    compile(project(":spring-context"))
    testImplementation 'org.junit.jupiter:junit-jupiter-api:5.7.0'
    testRuntimeOnly 'org.junit.jupiter:junit-jupiter-engine:5.7.0'
}

test {
    useJUnitPlatform()
}
```

## 参考资料

版本： `spring-framework v5.2.15.RELEASE`

idea 插件：

- Sequence Diagram
- idea-sourcetrail
