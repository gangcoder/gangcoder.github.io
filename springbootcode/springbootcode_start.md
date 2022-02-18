<!-- ---
title: SpringBoot Code 源码环境搭建
date: 2021-06-30 12:40:25
category: java100, springbootcode
--- -->

# SpringBoot Code 源码环境搭建

## 环境依赖

JDK 1.8+

Maven3.5+

SpringBoot v2.2.9 release

https://codeload.github.com/spring-projects/spring-boot/tar.gz/refs/tags/v2.2.9.RELEASE


```
// spring-boot
git clone https://github.com/spring-projects/spring-boot.git spring-projects/spring-boot


// spring
git clone https://github.com/spring-projects/spring-framework.git spring-projects/spring-framework
```

### 编译打包

编译源码：

```
mvn clean install -DskipTests -Pfast
```


### skipTest

```
mvn install -Dmaven.test.skip=true -DskipTests
```

spring-boot-build pom.xml 调整:

```xml
<properties>
    <disable.check>true</disable.check>
</properties>

<build>
    <plugins>                    
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-surefire-plugin</artifactId>
            <configuration>
                <skip>true</skip>
            </configuration>
        </plugin>
    </plugins>
</build>
```

## 新建测试 module

新建module，创建控制器和Action。

### pom 依赖

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
		 xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
	<modelVersion>4.0.0</modelVersion>
	<parent>
		<groupId>org.springframework.boot</groupId>
		<artifactId>spring-boot-starter-parent</artifactId>
		<version>2.2.9.RELEASE</version>
		<relativePath/> <!-- lookup parent from repository -->
	</parent>
	<groupId>com.coderead</groupId>
	<artifactId>bootsrv</artifactId>
	<version>0.0.1-SNAPSHOT</version>
	<name>bootsrv</name>
	<description>Demo project for Spring Boot</description>
	<properties>
		<java.version>1.8</java.version>
	</properties>
	<dependencies>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-web</artifactId>
		</dependency>

		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-test</artifactId>
			<scope>test</scope>
			<exclusions>
				<exclusion>
					<groupId>org.junit.vintage</groupId>
					<artifactId>junit-vintage-engine</artifactId>
				</exclusion>
			</exclusions>
		</dependency>
	</dependencies>

	<build>
		<plugins>
			<plugin>
				<groupId>org.springframework.boot</groupId>
				<artifactId>spring-boot-maven-plugin</artifactId>
			</plugin>
		</plugins>
	</build>

</project>
```

### 启动器代码

```java
package com.coderead.bootsrv;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class BootsrvApplication {

	public static void main(String[] args) {
		SpringApplication.run(BootsrvApplication.class, args);
	}

}
```

### 控制器代码

```java
package com.coderead.bootsrv.controller;

import com.coderead.bootsrv.service.WorldService;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

import javax.annotation.Resource;

@RestController
public class HelloController {

	@Resource
	private WorldService worldService;


	@GetMapping("/hello")
	public String hello() {
		String ret = "hello " + worldService.world();
		System.out.println(ret);

		return ret;
	}
}
```

## 参考资料

- []()

