<!-- ---
title: SpringBoot Code SpringApplication 初始化过程
date: 2021-06-30 12:40:28
category: java100, springbootcode
--- -->

# SpringBoot Code SpringApplication 初始化过程

![](draw/springbootcode_springapplication.svg)


构造 SpringApplication 实例，并运行它的run方法。

## 1. SpringApplication 类

```java
public class SpringApplication {

    // 构造 SpringApplication 实例
	public SpringApplication(ResourceLoader resourceLoader, Class<?>... primarySources) {
		// ...
		this.primarySources = new LinkedHashSet<>(Arrays.asList(primarySources));

        // ...
		this.webApplicationType = WebApplicationType.deduceFromClasspath();

        // ...
		setInitializers((Collection) getSpringFactoriesInstances(ApplicationContextInitializer.class));
		setListeners((Collection) getSpringFactoriesInstances(ApplicationListener.class));
		this.mainApplicationClass = deduceMainApplicationClass();
	}
}
```

### 1.1 设置初始化器 Initializer

```java
public class SpringApplication {
	private <T> Collection<T> getSpringFactoriesInstances(Class<T> type) {
		return getSpringFactoriesInstances(type, new Class<?>[] {});
	}

	private <T> Collection<T> getSpringFactoriesInstances(Class<T> type, Class<?>[] parameterTypes, Object... args) {
		ClassLoader classLoader = getClassLoader();
		// 从 spring.factories 加载配置
		Set<String> names = new LinkedHashSet<>(SpringFactoriesLoader.loadFactoryNames(type, classLoader));

        // 创建配置类实例
		List<T> instances = createSpringFactoriesInstances(type, parameterTypes, classLoader, args, names);
		AnnotationAwareOrderComparator.sort(instances);
		return instances;
	}

	private <T> List<T> createSpringFactoriesInstances(Class<T> type, Class<?>[] parameterTypes,
			ClassLoader classLoader, Object[] args, Set<String> names) {
        // ...

        for (String name : names) {
            // ...
            Class<?> instanceClass = ClassUtils.forName(name, classLoader);
            Constructor<?> constructor = instanceClass.getDeclaredConstructor(parameterTypes);

            T instance = (T) BeanUtils.instantiateClass(constructor, args);
            instances.add(instance);
        }
        
        // ...
		return instances;
	}
}
```

```java
public final class SpringFactoriesLoader {
	public static List<String> loadFactoryNames(Class<?> factoryType, @Nullable ClassLoader classLoader) {
		String factoryTypeName = factoryType.getName();
		return loadSpringFactories(classLoader).getOrDefault(factoryTypeName, Collections.emptyList());
	}

	private static Map<String, List<String>> loadSpringFactories(@Nullable ClassLoader classLoader) {
        // ...
        Enumeration<URL> urls = (classLoader != null ?
                classLoader.getResources(FACTORIES_RESOURCE_LOCATION) :
                ClassLoader.getSystemResources(FACTORIES_RESOURCE_LOCATION));
        
        result = new LinkedMultiValueMap<>();
        while (urls.hasMoreElements()) {
            URL url = urls.nextElement();
            UrlResource resource = new UrlResource(url);

            Properties properties = PropertiesLoaderUtils.loadProperties(resource);
            for (Map.Entry<?, ?> entry : properties.entrySet()) {
                String factoryTypeName = ((String) entry.getKey()).trim();

                for (String factoryImplementationName : StringUtils.commaDelimitedListToStringArray((String) entry.getValue())) {
                    result.add(factoryTypeName, factoryImplementationName.trim());
                }
            }
        }

        // ...
        return result;
	}
}
```

## 2. run 方法

run 方法内执行逻辑主要为：

1. 获取并启动监听器
2. 准备 environment，并加载配置文件
3. 创建 Spring 容器
4. Spring 容器前置处理
5. 刷新容器
6. 执行 Runners

```java
public class SpringApplication {
	public ConfigurableApplicationContext run(String... args) {
		
        // ...

        // 获取并启动监听器
		SpringApplicationRunListeners listeners = getRunListeners(args);
		listeners.starting();
		
        try {

            // environment 处理
            ConfigurableEnvironment environment = prepareEnvironment(listeners, applicationArguments);
			configureIgnoreBeanInfo(environment);

            // ...
			
            // 创建 Spring 容器 AnnotationConfigServletWebServerApplicationContext
            context = createApplicationContext();
			
            // Spring 容器前置处理
			prepareContext(context, environment, listeners, applicationArguments, printedBanner);
			
            // 刷新容器
            refreshContext(context);
			
            // 执行 Runners
			callRunners(context, applicationArguments);
		}
		catch (Throwable ex) {
			// ...
		}
	}
}
```

### 2.1 获取并启动监听器

SpringBoot 会在运行过程中的不同阶段，发送各种事件，来执行对应监听器的对应方法。

```java
public class SpringApplication {
	private SpringApplicationRunListeners getRunListeners(String[] args) {
		Class<?>[] types = new Class<?>[] { SpringApplication.class, String[].class };
		return new SpringApplicationRunListeners(logger,
				getSpringFactoriesInstances(SpringApplicationRunListener.class, types, this, args));
	}
}
```

```java
class SpringApplicationRunListeners {

	private final List<SpringApplicationRunListener> listeners;

	SpringApplicationRunListeners(Log log, Collection<? extends SpringApplicationRunListener> listeners) {
		this.log = log;
		this.listeners = new ArrayList<>(listeners);
	}

	void starting() {
		for (SpringApplicationRunListener listener : this.listeners) {
			listener.starting();
		}
	}
}
```

```java
public class EventPublishingRunListener implements SpringApplicationRunListener, Ordered {
	@Override
	public void starting() {
		this.initialMulticaster.multicastEvent(new ApplicationStartingEvent(this.application, this.args));
	}
}
```

```java
public class SimpleApplicationEventMulticaster extends AbstractApplicationEventMulticaster {
	@Override
	public void multicastEvent(final ApplicationEvent event, @Nullable ResolvableType eventType) {
		// ...
		for (ApplicationListener<?> listener : getApplicationListeners(event, type)) {
            // ...
            invokeListener(listener, event);
		}
	}
}
```

### 2.2 准备 environment

```java
public class SpringApplication {
	private ConfigurableEnvironment prepareEnvironment(SpringApplicationRunListeners listeners,
			ApplicationArguments applicationArguments) {
		// ...

        // 创建 environment
		ConfigurableEnvironment environment = getOrCreateEnvironment();
		
        // 触发 environment 配置时间，基于该事件，会读取配置文件
		listeners.environmentPrepared(environment);
		
        // ...
		return environment;
	}
}
```

### 2.3 Spring 容器前置处理

在容器刷新之前的准备动作。关键的操作：将启动类注入容器，为后续开启自动化配置奠定基础。

```java
public class SpringApplication {
	private void prepareContext(ConfigurableApplicationContext context, ConfigurableEnvironment environment,
			SpringApplicationRunListeners listeners, ApplicationArguments applicationArguments, Banner printedBanner) {
		// ...
        // 执行容器中的 ApplicationContextInitializer
		applyInitializers(context);
		
		// 获取启动类指定的参数
		Set<Object> sources = getAllSources();
		load(context, sources.toArray(new Object[0]));

        // ...
		listeners.contextLoaded(context);
	}

    // 调用初始化器
	protected void applyInitializers(ConfigurableApplicationContext context) {
		for (ApplicationContextInitializer initializer : getInitializers()) {
			// ...
			initializer.initialize(context);
		}
	}

    // 加载启动类
	public Set<Object> getAllSources() {
		Set<Object> allSources = new LinkedHashSet<>();
		if (!CollectionUtils.isEmpty(this.primarySources)) {
			allSources.addAll(this.primarySources);
		}
		
        // ...
		return Collections.unmodifiableSet(allSources);
	}

	protected void load(ApplicationContext context, Object[] sources) {
		// ...
		BeanDefinitionLoader loader = createBeanDefinitionLoader(getBeanDefinitionRegistry(context), sources);
		
        // ...
		loader.load();
	}
}
```

```java
class BeanDefinitionLoader {
	private int load(Class<?> source) {
		// ...
		if (isComponent(source)) {
			this.annotatedReader.register(source);
			return 1;
		}

        // ...
	}

	private boolean isComponent(Class<?> type) {
		if (MergedAnnotations.from(type, SearchStrategy.TYPE_HIERARCHY).isPresent(Component.class)) {
			return true;
		}
		// ...
	}
}
```

### 2.4 执行Runners

```java
public class SpringApplication {
	private void callRunners(ApplicationContext context, ApplicationArguments args) {
		List<Object> runners = new ArrayList<>();
		runners.addAll(context.getBeansOfType(ApplicationRunner.class).values());
		runners.addAll(context.getBeansOfType(CommandLineRunner.class).values());
		AnnotationAwareOrderComparator.sort(runners);

		for (Object runner : new LinkedHashSet<>(runners)) {
			if (runner instanceof ApplicationRunner) {
				callRunner((ApplicationRunner) runner, args);
			}
			// ...
		}
	}

	private void callRunner(ApplicationRunner runner, ApplicationArguments args) {
        // ...
		(runner).run(args);
	}

}
```

 

## 参考资料

- [SpringBoot 源码解析 （二）----- Spring Boot精髓：启动流程源码分析](https://www.cnblogs.com/java-chen-hao/p/11829344.html)

