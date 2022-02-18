<!-- ---
title: SpringBoot Code 依赖管理-spring-boot-starter-parent
date: 2021-06-30 12:40:26
category: java100, springbootcode
--- -->

# SpringBoot Code 依赖管理 spring-boot-starter-parent

## module 添加父依赖

```xml
// 父依赖
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>2.2.9.RELEASE</version>
    <relativePath/> <!-- lookup parent from repository -->
</parent>
```

## spring-boot-starter-parent

spring-boot-starter-parent 组成:

```xml
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-dependencies</artifactId>
    <version>${revision}</version>
    <relativePath>../../spring-boot-dependencies</relativePath>
</parent>
```

## spring-boot-dependencies

最终依赖 spring-boot-dependencies:

```xml
// 父依赖
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-build</artifactId>
    <version>${revision}</version>
    <relativePath>../..</relativePath>
</parent>
<artifactId>spring-boot-dependencies</artifactId>

// 依赖管理
<dependencyManagement>
    <dependencies>
        <!-- Spring Boot -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot</artifactId>
            <version>${revision}</version>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-test</artifactId>
            <version>${revision}</version>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-test-autoconfigure</artifactId>
            <version>${revision}</version>
        </dependency>
    </dependencies>
</dependencyManagement>
```


## spring-boot-starter-web 依赖

spring-boot-starter-web 依赖定义:

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-json</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-tomcat</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-validation</artifactId>
        <exclusions>
            <exclusion>
                <groupId>org.apache.tomcat.embed</groupId>
                <artifactId>tomcat-embed-el</artifactId>
            </exclusion>
        </exclusions>
    </dependency>
    <dependency>
        <groupId>org.springframework</groupId>
        <artifactId>spring-web</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework</groupId>
        <artifactId>spring-webmvc</artifactId>
    </dependency>
</dependencies>
```


## 参考资料

- []()