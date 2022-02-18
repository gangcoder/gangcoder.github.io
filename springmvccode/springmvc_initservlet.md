<!-- ---
title: DispatcherServlet 初始化
date: 2022-01-23 15:50:12
category: java100, springmvc, code
--- -->

# DispatcherServlet 初始化

DispatcherServlet 是 spring-mvc 的核心，该类进行真正逻辑实现。

## 1. 容器初始化

servlet 是一个 Java 编写的程序，基于 Http 协议。servlet 的生命周期是由 servlet 的容器来控制，分为三个阶段：初始化、运行和销毁。

在 servlet 初始化阶段会调用其 init 方法。

```java
public abstract class HttpServletBean extends HttpServlet implements EnvironmentCapable, EnvironmentAware {
    @Override
	public final void init() throws ServletException {
        // 解析 init-param 并封装到 pvs 变量中
		PropertyValues pvs = new ServletConfigPropertyValues(getServletConfig(), this.requiredProperties);
		if (!pvs.isEmpty()) {
			try {
				BeanWrapper bw = PropertyAccessorFactory.forBeanPropertyAccess(this);
                // ...
                // 进行属性注入
				bw.setPropertyValues(pvs, true);
			}
			catch (BeansException ex) {
				// ...
			}
		}

        // 初始化 servletBean
		initServletBean();
    }
}
```

初始化工作最终是创建或刷新 WebApplicationContext 实例并对 servlet 功能所使用的变量进行初始化。

```java
public abstract class FrameworkServlet extends HttpServletBean implements ApplicationContextAware {
	@Override
	protected final void initServletBean() throws ServletException {
        // ...
        this.webApplicationContext = initWebApplicationContext();
        initFrameworkServlet();

        // ...
    }

	protected WebApplicationContext initWebApplicationContext() {
        // ...
        // 创建一个 WebApplicationContext
        wac = createWebApplicationContext(rootContext);
    }

	protected WebApplicationContext createWebApplicationContext(@Nullable ApplicationContext parent) {
        // 默认是 XmlWebApplicationContext
		Class<?> contextClass = getContextClass();
		
        // ...
        // 通过反射来创建 contextClass
		ConfigurableWebApplicationContext wac = (ConfigurableWebApplicationContext) BeanUtils.instantiateClass(contextClass);

		// ...
        // spring beans 配置地址，用于后续spring 加载配置文件
		String configLocation = getContextConfigLocation();
		if (configLocation != null) {
			wac.setConfigLocation(configLocation);
		}

        // ...
		configureAndRefreshWebApplicationContext(wac);

		return wac;
	}

	protected void configureAndRefreshWebApplicationContext(ConfigurableWebApplicationContext wac) {
        // ...
        // 遍历 ApplicationContextInitializer，执行 initialize 方法
        applyInitializers(wac);

        // spring 核心刷新方法
		wac.refresh();
    }
}
```

## 2. mvc 初始化

Spring 容器refresh 刷新处理完成后，触发ContextRefreshedEvent 时间，调用 DispatcherServlet onRefresh 方法，用来刷新 Spring 在 Web 功能实现中所必须用到的全局变量。

```java
public class DispatcherServlet extends FrameworkServlet {
	@Override
	protected void onRefresh(ApplicationContext context) {
		initStrategies(context);
	}

	protected void initStrategies(ApplicationContext context) {
        // ...
        initLocaleResolver(context);
		initHandlerMappings(context);
		initHandlerAdapters(context);
        // ...
	}


}
```

实现上会先寻找用户自定义配置，没有找到，使用默认配置。

```java
public class DispatcherServlet extends FrameworkServlet {
	private void initLocaleResolver(ApplicationContext context) {
		try {
			this.localeResolver = context.getBean(LOCALE_RESOLVER_BEAN_NAME, LocaleResolver.class);
            // ...
		}
		catch (NoSuchBeanDefinitionException ex) {
			// 找不到处理bean 时，基于默认策略初始化
			this.localeResolver = getDefaultStrategy(context, LocaleResolver.class);
			// ...
		}
	}

	protected <T> T getDefaultStrategy(ApplicationContext context, Class<T> strategyInterface) {
        // 取默认策略
        // 默认策略文件 DispatcherServlet.properties
		List<T> strategies = getDefaultStrategies(context, strategyInterface);
		// ...
		return strategies.get(0);
	}

	private void initHandlerMappings(ApplicationContext context) {
		if (this.detectAllHandlerMappings) {
			// 找到所有 HandlerMapping 处理类
			Map<String, HandlerMapping> matchingBeans = BeanFactoryUtils.beansOfTypeIncludingAncestors(context, HandlerMapping.class, true, false);
			if (!matchingBeans.isEmpty()) {
				this.handlerMappings = new ArrayList<>(matchingBeans.values());
				// 排序
				AnnotationAwareOrderComparator.sort(this.handlerMappings);
			}
		}
		else {
			// ...
		}
	}

    private void initHandlerAdapters(ApplicationContext context) {
        if (this.detectAllHandlerAdapters) {
			// 找到所有处理适配器 
			Map<String, HandlerAdapter> matchingBeans =
					BeanFactoryUtils.beansOfTypeIncludingAncestors(context, HandlerAdapter.class, true, false);
			if (!matchingBeans.isEmpty()) {
				this.handlerAdapters = new ArrayList<>(matchingBeans.values());
				// 排序
				AnnotationAwareOrderComparator.sort(this.handlerAdapters);
			}
		}
		else {
            // ...
        }
    }
}
```


## 参考资料

- [Spring 源码学习(十) Spring mvc](http://www.justdojava.com/2019/07/21/spring-analysis-note-10)

