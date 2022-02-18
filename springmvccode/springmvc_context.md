<!-- ---
title: 初始化 WebApplicationContext
date: 2022-01-23 15:49:12
category: java100, springmvc, code
--- -->

# 初始化 WebApplicationContext

使用监听器加载 applicationContext 文件，在启动 web 容器时，自动装载 ApplicationContext 的配置信息。

## 1. ContextLoaderListener

ContextLoaderListener 实现了 ServletContextListener 这个接口，只要在 web.xml 配置了这个监听器，容器在启动时，就会执行 `contextInitialized(ServletContextEvent)` 这个方法，进行应用上下文初始化。

```java
public class ContextLoaderListener extends ContextLoader implements ServletContextListener {
    @Override
	public void contextInitialized(ServletContextEvent event) {
		initWebApplicationContext(event.getServletContext());
	}
}
```

实现的步骤如下：

1. 创建 WebApplicationContext 实例： createWebApplicationContext(servletContext);
2. 将实例记录在 servletContext 中

```java
public class ContextLoader {
	public WebApplicationContext initWebApplicationContext(ServletContext servletContext) {
        // ...
        if (this.context == null) {
            // 初始化 context
            this.context = createWebApplicationContext(servletContext);
        }
        if (this.context instanceof ConfigurableWebApplicationContext) {
            ConfigurableWebApplicationContext cwac = (ConfigurableWebApplicationContext) this.context;
            if (!cwac.isActive()) {
                // 配置和刷新 context
                configureAndRefreshWebApplicationContext(cwac, servletContext);
            }
        }

        // 记录在 ServletContext 中
        servletContext.setAttribute(WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE, this.context);

        // ...
        return this.context;
    }
}
```

创建 WebApplicationContext 实例，实际是 XmlWebApplicationContext。

```java
public class ContextLoader {
    private static final Properties defaultStrategies;

	protected WebApplicationContext createWebApplicationContext(ServletContext sc) {
		Class<?> contextClass = determineContextClass(sc);

        // 初始化实例
		return (ConfigurableWebApplicationContext) BeanUtils.instantiateClass(contextClass);
	}

	protected Class<?> determineContextClass(ServletContext servletContext) {
		if (contextClassName != null) {
			// ...
		}
		else {
			contextClassName = defaultStrategies.getProperty(WebApplicationContext.class.getName());
			try {
                // org.springframework.web.context.support.XmlWebApplicationContext
				return ClassUtils.forName(contextClassName, ContextLoader.class.getClassLoader());
			}
			catch (ClassNotFoundException ex) {
				// ...
			}
		}
	}
}
```

## 2. refresh

刷新 context。

```java
public class ContextLoader {
    protected void configureAndRefreshWebApplicationContext(ConfigurableWebApplicationContext wac, ServletContext sc) {
        // 设置 ServletContext 
        wac.setServletContext(sc);

        // spring beans 配置文件地址
		String configLocationParam = sc.getInitParameter(CONFIG_LOCATION_PARAM);
		if (configLocationParam != null) {
			wac.setConfigLocation(configLocationParam);
		}

		// context 自定义初始化处理
		customizeContext(sc, wac);

        // 刷新 spring application context
		wac.refresh();
	}
}
```


## 参考资料

- [Spring 源码学习(十) Spring mvc](http://www.justdojava.com/2019/07/21/spring-analysis-note-10)

