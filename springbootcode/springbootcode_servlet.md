<!-- ---
title: ServletWebServer 自动配置分析
date: 2022-02-11 13:34:35
category: java100, springboot, code
--- -->

# ServletWebServer 自动配置分析

`ServletWebServerFactoryAutoConfiguration` 用于自动配置 Servlet，通过自动配置导入。

## Servlet 容器自动配置原理

根据条件，自动配置 Servlet 容器。

```java
@Import({ ServletWebServerFactoryAutoConfiguration.BeanPostProcessorsRegistrar.class,
		ServletWebServerFactoryConfiguration.EmbeddedTomcat.class,
		ServletWebServerFactoryConfiguration.EmbeddedJetty.class,
		ServletWebServerFactoryConfiguration.EmbeddedUndertow.class })
public class ServletWebServerFactoryAutoConfiguration {
}
```

根据条件，导入 EmbeddedTomcat。

```java
@Configuration(proxyBeanMethods = false)
@ConditionalOnClass({ Servlet.class, Tomcat.class, UpgradeProtocol.class })
@ConditionalOnMissingBean(value = ServletWebServerFactory.class, search = SearchStrategy.CURRENT)
static class EmbeddedTomcat {
    @Bean
    TomcatServletWebServerFactory tomcatServletWebServerFactory(
            ObjectProvider<TomcatConnectorCustomizer> connectorCustomizers,
            ObjectProvider<TomcatContextCustomizer> contextCustomizers,
            ObjectProvider<TomcatProtocolHandlerCustomizer<?>> protocolHandlerCustomizers) {

        // 创建 TomcatServletWebServer 工厂实例
        TomcatServletWebServerFactory factory = new TomcatServletWebServerFactory();
        
        // ...
        return factory;
    }
}
```

TomcatServletWebServerFactory 实现 ServletWebServerFactory 接口，用于获取容器实例。

```java
public class TomcatServletWebServerFactory extends AbstractServletWebServerFactory
		implements ConfigurableTomcatWebServerFactory, ResourceLoaderAware {
	@Override
	public WebServer getWebServer(ServletContextInitializer... initializers) {
		// ...
		Tomcat tomcat = new Tomcat();
		
        // ...
		tomcat.getHost().setAutoDeploy(false);
		configureEngine(tomcat.getEngine());
		
        // ...
		prepareContext(tomcat.getHost(), initializers);
		return getTomcatWebServer(tomcat);
	}

	protected TomcatWebServer getTomcatWebServer(Tomcat tomcat) {
		return new TomcatWebServer(tomcat, getPort() >= 0);
	}
}
```

Servlet 容器启动：

```java
public class TomcatWebServer implements WebServer {
	public TomcatWebServer(Tomcat tomcat, boolean autoStart) {
		// ...
		this.tomcat = tomcat;
		this.autoStart = autoStart;
		initialize();
	}

	private void initialize() throws WebServerException {     
        // ...
        // 启动容器
        this.tomcat.start();

        // ...
	}
}
```

## Servlet 容器启动触发

刷新容器时，触发 ServletWebServerApplicationContext 的 onRefresh 方法，创建 Server。

```java
public class ServletWebServerApplicationContext extends GenericWebApplicationContext
		implements ConfigurableWebServerApplicationContext {
	@Override
	protected void onRefresh() {
		super.onRefresh();

        // ...
        createWebServer();
	}

	private void createWebServer() {
		// ...

        // 获取 TomcatServletWebServer 工厂实例
        ServletWebServerFactory factory = getWebServerFactory();

        // 创建 WebServer
        this.webServer = factory.getWebServer(getSelfInitializer());

        // ...
	}

	protected ServletWebServerFactory getWebServerFactory() {
		String[] beanNames = getBeanFactory().getBeanNamesForType(ServletWebServerFactory.class);
		
        // ...
		return getBeanFactory().getBean(beanNames[0], ServletWebServerFactory.class);
	}
}
```


## 参考资料

- [SpringBoot 源码解析 （六）----- Spring Boot的核心能力 - 内置Servlet容器源码分析（Tomcat）](https://www.cnblogs.com/java-chen-hao/p/11837057.html)
