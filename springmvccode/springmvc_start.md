<!-- ---
title: Spring MVC 环境搭建
date: 2022-01-11 12:07:56
category: java100, spring, springcode
--- -->

# Spring MVC 环境搭建

下载并配置 tomcat，然后准备 spring mvc 环境。

## gradle 依赖

build.gradle 依赖配置

```gradle
plugins {
    id 'java'
    id 'war'
}

group 'org.springframework'
version '5.2.15.RELEASE'

repositories {
    mavenCentral()
}

dependencies {
    compile(project(":spring-context"))
    compile(project(":spring-webmvc"))
    compile(project(":spring-test"))

    compile("javax.servlet:javax.servlet-api")
    compile("javax.servlet:jstl:1.2")
    compile("javax.servlet.jsp:javax.servlet.jsp-api")

    testImplementation 'junit:junit:4.11'
    testImplementation 'org.junit.jupiter:junit-jupiter-api:5.7.0'
    testRuntimeOnly 'org.junit.jupiter:junit-jupiter-engine:5.7.0'
}

test {
    useJUnitPlatform()
}
```

## 1. 基于xml 配置

### 1.1 配置web.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<web-app xmlns="http://xmlns.jcp.org/xml/ns/javaee"
		 xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
		 xsi:schemaLocation="http://xmlns.jcp.org/xml/ns/javaee http://xmlns.jcp.org/xml/ns/javaee/web-app_3_1.xsd"
		 version="3.1">
	<!-- 使用 ContextLoaderListener时,告诉它 Spring 配置文件地址 -->
	<context-param>
		<param-name>contextConfigLocation</param-name>
		<param-value>/WEB-INF/applicationContext.xml</param-value>
	</context-param>

	<!-- 使用监听器加载 applicationContext 文件 -->
	<listener>
		<listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>
	</listener>

	<!-- 配置 DispatcherServlet -->
	<servlet>
		<servlet-name>dispatcherServlet</servlet-name>
		<servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
		<init-param>
			<param-name>contextConfigLocation</param-name>
			<param-value>/WEB-INF/spring-mvc.xml</param-value>
		</init-param>
	</servlet>
	<servlet-mapping>
		<servlet-name>dispatcherServlet</servlet-name>
		<url-pattern>/</url-pattern>
	</servlet-mapping>
</web-app>
```

### 1.2 配置 spring-mvc.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
	   xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns:mvc="http://www.springframework.org/schema/mvc"
	   xmlns:context="http://www.springframework.org/schema/context"
	   xsi:schemaLocation="http://www.springframework.org/schema/beans
       http://www.springframework.org/schema/beans/spring-beans.xsd http://www.springframework.org/schema/mvc
       http://www.springframework.org/schema/mvc/spring-mvc.xsd http://www.springframework.org/schema/context
       http://www.springframework.org/schema/context/spring-context.xsd">

	<!--扫描包,自动注入bean-->
	<context:component-scan base-package="web.controller"/>
	<!--使用注解开发spring mvc-->
	<mvc:annotation-driven/>

	<!--视图解析器-->
	<bean class="org.springframework.web.servlet.view.InternalResourceViewResolver">
		<property name="prefix" value="/WEB-INF/views/"/>
		<property name="suffix" value=".jsp"/>
	</bean>

</beans>
```

### 1.3 配置 applicationContext.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
	   xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	   xmlns:context="http://www.springframework.org/schema/context"
	   xsi:schemaLocation="http://www.springframework.org/schema/beans
       http://www.springframework.org/schema/beans/spring-beans.xsd
       http://www.springframework.org/schema/context
       http://www.springframework.org/schema/context/spring-context.xsd">

	<!--这里比较简单，只是通知 Spring 扫描对应包下的 bean -->
	<context:component-scan base-package="web.controller"/>

</beans>
```

## 2. 基于代码配置

```java
package com.explorejava.springmvc.javaconfig;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.ComponentScan;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.servlet.config.annotation.DefaultServletHandlerConfigurer;
import org.springframework.web.servlet.config.annotation.EnableWebMvc;
import org.springframework.web.servlet.config.annotation.WebMvcConfigurer;
import org.springframework.web.servlet.view.InternalResourceViewResolver;


@EnableWebMvc
@Configuration
@ComponentScan(basePackages = "com.explorejava.springmvc")
public class ApplicationConfigurerAdapter implements WebMvcConfigurer {

	@Override
	public void configureDefaultServletHandling(DefaultServletHandlerConfigurer configurer) {
		configurer.enable();
	}

	@Bean
	public InternalResourceViewResolver viewResolver() {
		InternalResourceViewResolver resolver = new InternalResourceViewResolver();
		resolver.setPrefix("/WEB-INF/jsp/");
		resolver.setSuffix(".jsp");
		return resolver;
	}

}
```

```java
package com.explorejava.springmvc.javaconfig;

import org.springframework.web.WebApplicationInitializer;
import org.springframework.web.context.support.AnnotationConfigWebApplicationContext;
import org.springframework.web.servlet.DispatcherServlet;

import javax.servlet.ServletContext;
import javax.servlet.ServletRegistration;

public class DispatcherServletConfigInWebXML implements WebApplicationInitializer {

	@Override
	public void onStartup(ServletContext container) {

		AnnotationConfigWebApplicationContext ctx = new AnnotationConfigWebApplicationContext();
		ctx.register(ApplicationConfigurerAdapter.class);
		ctx.setServletContext(container);

		ServletRegistration.Dynamic servlet = container.addServlet("dispatcher", new DispatcherServlet(ctx));

		servlet.setLoadOnStartup(1);
		servlet.addMapping("/example/*");
	}
}
```

配置 WEB-INF 下 web.xml 文件:

```xml
<web-app>
  <display-name>Archetype Created Web Application</display-name>
</web-app>
```

## 3. 编写 controller

```java
package com.explorejava.springmvc.helloworld;

import org.springframework.stereotype.Controller;
import org.springframework.ui.ModelMap;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestMethod;
import org.springframework.web.bind.annotation.RequestParam;
import org.springframework.web.servlet.ModelAndView;

@Controller
public class HelloWorldController {
	@RequestMapping(value = "/helloModel", method = RequestMethod.GET)
	public ModelAndView hello() {
		ModelAndView modelAndView = new ModelAndView("helloworld")
				.addObject("name", "Ramesh");
		return modelAndView;
	}
	
	@RequestMapping(value = "/hello/{userName}", method = RequestMethod.GET)
	public String sayHello(@PathVariable String userName , ModelMap model) {
		model.addAttribute("name", "Hello "+userName+" from Spring 4 MVC");
		return "helloworld";
	}

	// http://localhost:8080/Gradle___org_springframework___mvc_debug_5_2_15_RELEASE_war/example/helloagain
	@RequestMapping(value = "/helloagain", method = RequestMethod.GET)
	public String sayHelloAgain(@RequestParam(required=false) String employeeName, ModelMap model) {
		model.addAttribute("greeting", "Hello World Again, from Spring 4 MVC "+ employeeName);
		System.out.println("welcome");
		return "welcome";
	}
}
```


## 参考资料

- maven spring mvc helloworld https://github.com/eldhoseak/spring-mvc-helloworld-example.git
- [SpringMVC的第一个环境的搭建](https://www.cnblogs.com/codegzy/p/15219886.html)
