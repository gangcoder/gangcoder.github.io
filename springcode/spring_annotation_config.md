<!-- ---
title: AnnotationConfig 注解驱动上下文实现
date: 2022-02-09 07:31:50
category: java100, spring, springcode
--- -->

# AnnotationConfig 注解驱动上下文实现

Spring 提供 AnnotationConfigApplicationContext 实现注解驱动开发。

示例代码：

```java
// 配置类
@Configuration
@ComponentScan(basePackages = "com.hello")
public class TestConfig {
}

// 主类
public class MyAnnoApplication {
	public static void main(String[] args) {
		ApplicationContext ctx = new AnnotationConfigApplicationContext(TestConfig.class);
	}
}
```

## 1. 初始化

```java
public class AnnotationConfigApplicationContext extends GenericApplicationContext implements AnnotationConfigRegistry {
	public AnnotationConfigApplicationContext(Class<?>... componentClasses) {
		this();

        // 注册bean配置类
		register(componentClasses);
		refresh();
	}

	public AnnotationConfigApplicationContext() {
        // 初始化bean 读取器和扫描器
		this.reader = new AnnotatedBeanDefinitionReader(this);
		this.scanner = new ClassPathBeanDefinitionScanner(this);
	}

	@Override
	public void register(Class<?>... componentClasses) {
		// ...
		this.reader.register(componentClasses);
	}
}
```

## 2. bean 读取器

读取指定配置类注解信息并注册 bean。

AnnotatedBeanDefinitionReader 在初始化 reader 时注册 `ConfigurationClassPostProcessor` 组件，在 refresh 时用来处理 `@Configuration` 注解的类。

```java
public class AnnotatedBeanDefinitionReader {
	public AnnotatedBeanDefinitionReader(BeanDefinitionRegistry registry) {
		this(registry, getOrCreateEnvironment(registry));
	}

	public AnnotatedBeanDefinitionReader(BeanDefinitionRegistry registry, Environment environment) {
		// ...
		this.conditionEvaluator = new ConditionEvaluator(registry, environment, null);
		AnnotationConfigUtils.registerAnnotationConfigProcessors(this.registry);
	}

	public void register(Class<?>... componentClasses) {
		for (Class<?> componentClass : componentClasses) {
			registerBean(componentClass);
		}
	}

	public void registerBean(Class<?> beanClass) {
		doRegisterBean(beanClass, null, null, null, null);
	}

	private <T> void doRegisterBean(Class<T> beanClass, @Nullable String name,
			@Nullable Class<? extends Annotation>[] qualifiers, @Nullable Supplier<T> supplier,
			@Nullable BeanDefinitionCustomizer[] customizers) {

        // 将Bean 配置类转成 AnnotatedGenericBeanDefinition
		AnnotatedGenericBeanDefinition abd = new AnnotatedGenericBeanDefinition(beanClass);
		
        // ...
        // 处理Lazy, primary DependsOn, Role ,Description 注解
		AnnotationConfigUtils.processCommonDefinitionAnnotations(abd);
		
        // ...
        // 自定义bean注册
		if (customizers != null) {
			for (BeanDefinitionCustomizer customizer : customizers) {
				customizer.customize(abd);
			}
		}

        // 注册 bean
		BeanDefinitionHolder definitionHolder = new BeanDefinitionHolder(abd, beanName);
		definitionHolder = AnnotationConfigUtils.applyScopedProxyMode(scopeMetadata, definitionHolder, this.registry);
		BeanDefinitionReaderUtils.registerBeanDefinition(definitionHolder, this.registry);
	}
}
```

```java
public abstract class AnnotationConfigUtils {
	public static void registerAnnotationConfigProcessors(BeanDefinitionRegistry registry) {
		registerAnnotationConfigProcessors(registry, null);
	}

	public static Set<BeanDefinitionHolder> registerAnnotationConfigProcessors(
			BeanDefinitionRegistry registry, @Nullable Object source) {
        // ...
		Set<BeanDefinitionHolder> beanDefs = new LinkedHashSet<>(8);

        // 注册 ConfigurationClassPostProcessor
		if (!registry.containsBeanDefinition(CONFIGURATION_ANNOTATION_PROCESSOR_BEAN_NAME)) {
			RootBeanDefinition def = new RootBeanDefinition(ConfigurationClassPostProcessor.class);
			def.setSource(source);
			beanDefs.add(registerPostProcessor(registry, def, CONFIGURATION_ANNOTATION_PROCESSOR_BEAN_NAME));
		}

		if (!registry.containsBeanDefinition(AUTOWIRED_ANNOTATION_PROCESSOR_BEAN_NAME)) {
			RootBeanDefinition def = new RootBeanDefinition(AutowiredAnnotationBeanPostProcessor.class);
			def.setSource(source);
			beanDefs.add(registerPostProcessor(registry, def, AUTOWIRED_ANNOTATION_PROCESSOR_BEAN_NAME));
		}

        // ...
		return beanDefs;
	}

	private static BeanDefinitionHolder registerPostProcessor(
			BeanDefinitionRegistry registry, RootBeanDefinition definition, String beanName) {

		definition.setRole(BeanDefinition.ROLE_INFRASTRUCTURE);
		registry.registerBeanDefinition(beanName, definition);
		return new BeanDefinitionHolder(definition, beanName);
	}

}
```

## 3. 刷新处理差异

AnnotationConfigApplicationContext 的 refresh 方法在AbstractApplicationContext 容器中实现。

但是 refreshBeanFactory 实现不同。

xml 方式，使用 AbstractRefreshableApplicationContext 容器中的实现。该容器中实现xml 配置文件定位，并通过BeanDefinition 载入和解析xml 配置文件。

注解方式，使用 GenericApplicationContext 实现。此时并没有解析项目包下的注解，而是通过 ConfigurationClassPostProcessor 后置处理器完成对bean 的加载。

```java
public class GenericApplicationContext extends AbstractApplicationContext implements BeanDefinitionRegistry {
	@Override
	protected final void refreshBeanFactory() throws IllegalStateException {
		// ...
		this.beanFactory.setSerializationId(getId());
	}
}
```


Spring容器在启动总结：

1. 先保存所有注册进来的Bean的定义信息
2. Spring容器根据条件创建Bean实例，区分单例，还是原型，后置处理器等
3. 通过后置处理器来增加bean 的功能。

## 参考资料

- [Spring源码解析 – AnnotationConfigApplicationContext容器创建过程](https://www.cnblogs.com/ashleyboy/p/9662119.html)
