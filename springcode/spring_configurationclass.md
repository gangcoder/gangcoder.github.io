<!-- ---
title: @Configuration 注解解析
date: 2022-02-09 12:01:18
category: java100, spring, springcode
--- -->

# @Configuration 注解解析

ConfigurationClassPostProcessor 实现 BeanDefinitionRegistryPostProcessor，所以会在容器初始化时优先执行，主要负责解析所有 `@Configuration` 标签类，并将`Bean 定义`注册到容器中。

## 1. postProcessBeanDefinitionRegistry 实现

postProcessBeanDefinitionRegistry 开始 Configuration 处理。

```java
public class ConfigurationClassPostProcessor implements BeanDefinitionRegistryPostProcessor,
		PriorityOrdered, ResourceLoaderAware, BeanClassLoaderAware, EnvironmentAware {
	@Override
	public void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) {
		// ...

        // 解析配置类 bean
		processConfigBeanDefinitions(registry);
	}

	public void processConfigBeanDefinitions(BeanDefinitionRegistry registry) {
		// ...

        // 所有已经注册的bean
		String[] candidateNames = registry.getBeanDefinitionNames();
		for (String beanName : candidateNames) {
			BeanDefinition beanDef = registry.getBeanDefinition(beanName);
			if (beanDef.getAttribute(ConfigurationClassUtils.CONFIGURATION_CLASS_ATTRIBUTE) != null) {
				// ...
			}
			else if (ConfigurationClassUtils.checkConfigurationClassCandidate(beanDef, this.metadataReaderFactory)) {
                // 检查是否是配置类
				configCandidates.add(new BeanDefinitionHolder(beanDef, beanName));
			}
		}

		// ...
        
        // 初始化ConfigurationClassParser解析器
		ConfigurationClassParser parser = new ConfigurationClassParser(
				this.metadataReaderFactory, this.problemReporter, this.environment,
				this.resourceLoader, this.componentScanBeanNameGenerator, registry);

		// ...
		do {
            // 解析配置类，这里会先解析为 configClass
			parser.parse(candidates);

			// ...

			// 获取所有待加载列表
			Set<ConfigurationClass> configClasses = new LinkedHashSet<>(parser.getConfigurationClasses());

			// 配置读取器
			if (this.reader == null) {
				this.reader = new ConfigurationClassBeanDefinitionReader(
						registry, this.sourceExtractor, this.resourceLoader, this.environment,
						this.importBeanNameGenerator, parser.getImportRegistry());
			}

            // 加载待加载列表配置
			// 基于 configClass 创建 bean定义
			// 主要实现将 @Configuration @Import @ImportResource @ImportRegistrar 定义的bean 注册到容器
			this.reader.loadBeanDefinitions(configClasses);

			// 导入的 bean 定义，实际会再被解析一次，用于递归解析导入bean 的 Configuration 配置
			// ...
		}
		while (!candidates.isEmpty());

        // ...
	}
}
```

## 2. ConfigurationClassParser

ConfigurationClassParser 实际解析 Configuration 配置类。

```java
class ConfigurationClassParser {
	public void parse(Set<BeanDefinitionHolder> configCandidates) {
		for (BeanDefinitionHolder holder : configCandidates) {
			BeanDefinition bd = holder.getBeanDefinition();
			try {
				if (bd instanceof AnnotatedBeanDefinition) {
					parse(((AnnotatedBeanDefinition) bd).getMetadata(), holder.getBeanName());
				}
				// ...
			}
			catch (BeanDefinitionStoreException ex) {
				// ...
			}
		}

		// 处理 DeferredImportSelector 接口实现类
		this.deferredImportSelectorHandler.process();
	}

	protected final void parse(AnnotationMetadata metadata, String beanName) throws IOException {
		processConfigurationClass(new ConfigurationClass(metadata, beanName), DEFAULT_EXCLUSION_FILTER);
	}

	protected void processConfigurationClass(ConfigurationClass configClass, Predicate<String> filter) throws IOException {
		// 检查是否需要解析
		if (this.conditionEvaluator.shouldSkip(configClass.getMetadata(), ConfigurationPhase.PARSE_CONFIGURATION)) {
			return;
		}

		// ...
		SourceClass sourceClass = asSourceClass(configClass, filter);
		do {
			// 解析配置类，解析出来的信息放到 configClass 中，后面再通过加载器加载
			sourceClass = doProcessConfigurationClass(configClass, sourceClass, filter);
		}
		while (sourceClass != null);

		this.configurationClasses.put(configClass, configClass);
	}

    // 从配置类中解析所有bean，包括处理内部类，父类以及各种注解
	protected final SourceClass doProcessConfigurationClass(
			ConfigurationClass configClass, SourceClass sourceClass, Predicate<String> filter)
			throws IOException {

		if (configClass.getMetadata().isAnnotated(Component.class.getName())) {
			// 递归处理嵌套类
			processMemberClasses(configClass, sourceClass, filter);
		}

		// ...

		// 处理 @ComponentScan 注解
		Set<AnnotationAttributes> componentScans = AnnotationConfigUtils.attributesForRepeatable(
				sourceClass.getMetadata(), ComponentScans.class, ComponentScan.class);
		if (!componentScans.isEmpty() &&
				!this.conditionEvaluator.shouldSkip(sourceClass.getMetadata(), ConfigurationPhase.REGISTER_BEAN)) {
			for (AnnotationAttributes componentScan : componentScans) {
				// 按@CmponentScan注解扫描bean
				Set<BeanDefinitionHolder> scannedBeanDefinitions =
						this.componentScanParser.parse(componentScan, sourceClass.getMetadata().getClassName());
				// ...
			}
		}

		// 处理@Import 注解
		processImports(configClass, sourceClass, getImports(sourceClass), filter, true);

		// 处理@Bean 注解修饰的方法
		Set<MethodMetadata> beanMethods = retrieveBeanMethodMetadata(sourceClass);
		for (MethodMetadata methodMetadata : beanMethods) {
			configClass.addBeanMethod(new BeanMethod(methodMetadata, configClass));
		}

		// ...
	}    
}
```

### 2.1 @ComponentScan 注解解析过程

@ComponentScan 注解解析通过调用 ComponentScanAnnotationParser 的 parse 方法完成。

parse() 方法最终通过调用 ClassPathBeanDefinitionScanner.doScan 方法实现扫描。

```java
class ComponentScanAnnotationParser {
	public Set<BeanDefinitionHolder> parse(AnnotationAttributes componentScan, final String declaringClass) {
		ClassPathBeanDefinitionScanner scanner = new ClassPathBeanDefinitionScanner(this.registry,
				componentScan.getBoolean("useDefaultFilters"), this.environment, this.resourceLoader);

		// ...
		Set<String> basePackages = new LinkedHashSet<>();
		String[] basePackagesArray = componentScan.getStringArray("basePackages");
		for (String pkg : basePackagesArray) {
			String[] tokenized = StringUtils.tokenizeToStringArray(this.environment.resolvePlaceholders(pkg),
					ConfigurableApplicationContext.CONFIG_LOCATION_DELIMITERS);
			Collections.addAll(basePackages, tokenized);
		}
		
		// ...
		return scanner.doScan(StringUtils.toStringArray(basePackages));
	}
}
```

### 2.2 @Bean 注解解析过程

```java
class ConfigurationClassParser {
	private Set<MethodMetadata> retrieveBeanMethodMetadata(SourceClass sourceClass) {
        AnnotationMetadata original = sourceClass.getMetadata();
        // 获取所有@Bean 注解的方法
		Set<MethodMetadata> beanMethods = original.getAnnotatedMethods(Bean.class.getName());
		// ...
		return beanMethods;
	}
}
```

```java
final class SimpleAnnotationMetadata implements AnnotationMetadata {
	@Override
	public Set<MethodMetadata> getAnnotatedMethods(String annotationName) {
		Set<MethodMetadata> annotatedMethods = null;
		for (MethodMetadata annotatedMethod : this.annotatedMethods) {
			if (annotatedMethod.isAnnotated(annotationName)) {
				if (annotatedMethods == null) {
					annotatedMethods = new LinkedHashSet<>(4);
				}
				annotatedMethods.add(annotatedMethod);
			}
		}
		return annotatedMethods != null ? annotatedMethods : Collections.emptySet();
	}
}
```

## 3. @Import 注解处理实现

```java
class ConfigurationClassParser {
	private void processImports(ConfigurationClass configClass, SourceClass currentSourceClass,
			Collection<SourceClass> importCandidates, Predicate<String> exclusionFilter,
			boolean checkForCircularImports) {

		// ...
		for (SourceClass candidate : importCandidates) {
			// 对import的内容进行分类

			// 实现 ImportSelector 接口的类
			if (candidate.isAssignable(ImportSelector.class)) {
				// 实例化导入的类
				ImportSelector selector = ParserStrategyUtils.instantiateClass(candidateClass, ImportSelector.class,
						this.environment, this.resourceLoader, this.registry);
				
				// 延迟加载的 ImportSelector，先放到List 中
				if (selector instanceof DeferredImportSelector) {
					this.deferredImportSelectorHandler.handle(configClass, (DeferredImportSelector) selector);
				}
				else {
					// 普通的 ImportSelector , 执行其selectImports方法, 获取需要导入的类的全限定类名数组
					String[] importClassNames = selector.selectImports(currentSourceClass.getMetadata());
					Collection<SourceClass> importSourceClasses = asSourceClasses(importClassNames, exclusionFilter);

					// 导入的普通类，迭代调用，会作为普通 configClass 加入待加载列表
					processImports(configClass, currentSourceClass, importSourceClasses, exclusionFilter, false);
				}
			}
			else if (candidate.isAssignable(ImportBeanDefinitionRegistrar.class)) {
				// 实现 ImportBeanDefinitionRegistrar
				// ...
				ImportBeanDefinitionRegistrar registrar =
						ParserStrategyUtils.instantiateClass(candidateClass, ImportBeanDefinitionRegistrar.class,
								this.environment, this.resourceLoader, this.registry);

				// 将ImportBeanDefinitionRegistrar 添加到configClass，最后加载时再调用
				configClass.addImportBeanDefinitionRegistrar(registrar, currentSourceClass.getMetadata());
			}
			else {
				// 普通类，通过 asConfigClass 生成新的 configClass，并将 configClass 加入待加载列表
				processConfigurationClass(candidate.asConfigClass(configClass), exclusionFilter);
			}
		}
	}
}
```

### 延迟 Import 实现

```java
class DeferredImportSelectorHandler {
	@Nullable
	private List<DeferredImportSelectorHolder> deferredImportSelectors = new ArrayList<>();

	// 延迟 Import 存入List
	public void handle(ConfigurationClass configClass, DeferredImportSelector importSelector) {
		DeferredImportSelectorHolder holder = new DeferredImportSelectorHolder(configClass, importSelector);
		// ...

		this.deferredImportSelectors.add(holder);
	}

	// 延迟 Import 处理
	public void process() {
		// ...
		DeferredImportSelectorGroupingHandler handler = new DeferredImportSelectorGroupingHandler();
		
		// ...
		deferredImports.forEach(handler::register);
		handler.processGroupImports();
	}
}
```

延迟 Import 处理导入处理：

```java
class DeferredImportSelectorGroupingHandler {

	private final Map<Object, DeferredImportSelectorGrouping> groupings = new LinkedHashMap<>();

	private final Map<AnnotationMetadata, ConfigurationClass> configurationClasses = new HashMap<>();

	public void register(DeferredImportSelectorHolder deferredImport) {
		// 获取 DeferredImportSelector 的 Group
		Class<? extends Group> group = deferredImport.getImportSelector().getImportGroup();

		// 创建Group 实例，并且放入 groupings
		DeferredImportSelectorGrouping grouping = this.groupings.computeIfAbsent(
				(group != null ? group : deferredImport),
				key -> new DeferredImportSelectorGrouping(createGroup(group)));
		// ...
	}

	public void processGroupImports() {
		// 遍历 groupings
		for (DeferredImportSelectorGrouping grouping : this.groupings.values()) {
			// ...

			// 获取需要导入的全类名
			grouping.getImports().forEach(entry -> {
				// ...

				// 取到引入的 entry.getImportClassName() 全类名，迭代调用 processImports，作为普通类，注册到待加载列表
				processImports(configurationClass, asSourceClass(configurationClass, exclusionFilter),
							Collections.singleton(asSourceClass(entry.getImportClassName(), exclusionFilter)),
							exclusionFilter, false);
			});
		}
	}

	private Group createGroup(@Nullable Class<? extends Group> type) {
		// 创建 Group 实例
		return ParserStrategyUtils.instantiateClass(effectiveType, Group.class,
				ConfigurationClassParser.this.environment,
				ConfigurationClassParser.this.resourceLoader,
				ConfigurationClassParser.this.registry);
	}
}
```

```java
static class DeferredImportSelectorGrouping {
	public Iterable<Group.Entry> getImports() {
		// ...

		// 调用 Group 的 selectImports，获取需要注册的全类名
		return this.group.selectImports();
	}
}
```

## 4. ConfigurationClassBeanDefinitionReader

ConfigurationClassBeanDefinitionReader 基于 configClass 创建 bean定义，并注册到容器。

主要是配置类中的 `@Bean`, `@Import` 注解的 bean 定义注册。

```java
class ConfigurationClassBeanDefinitionReader {
	public void loadBeanDefinitions(Set<ConfigurationClass> configurationModel) {
		TrackedConditionEvaluator trackedConditionEvaluator = new TrackedConditionEvaluator();
		for (ConfigurationClass configClass : configurationModel) {
			loadBeanDefinitionsForConfigurationClass(configClass, trackedConditionEvaluator);
		}
	}

	private void loadBeanDefinitionsForConfigurationClass(
			ConfigurationClass configClass, TrackedConditionEvaluator trackedConditionEvaluator) {
        // ...

		// 通过 Configuration 引入的类，会通过该步骤注册，包括 Import 引入的普通类和 ImportSelector 实现类
		if (configClass.isImported()) {
			registerBeanDefinitionForImportedConfigurationClass(configClass);
		}

		// @Bean 注解的 bean，通过该步骤注册
		for (BeanMethod beanMethod : configClass.getBeanMethods()) {
			loadBeanDefinitionsForBeanMethod(beanMethod);
		}

        // ...

        // ImportBeanDefinitionRegistrars 指定的资源注册为bean
		loadBeanDefinitionsFromRegistrars(configClass.getImportBeanDefinitionRegistrars());
	}

	private void registerBeanDefinitionForImportedConfigurationClass(ConfigurationClass configClass) {
		// ...
		AnnotatedGenericBeanDefinition configBeanDef = new AnnotatedGenericBeanDefinition(metadata);

		// 解析Bean 描述注解
		AnnotationConfigUtils.processCommonDefinitionAnnotations(configBeanDef, metadata);

		// BeanDefinition
		BeanDefinitionHolder definitionHolder = new BeanDefinitionHolder(configBeanDef, configBeanName);
		
		// ...

		// 注册
		this.registry.registerBeanDefinition(definitionHolder.getBeanName(), definitionHolder.getBeanDefinition());
	}

	private void loadBeanDefinitionsForBeanMethod(BeanMethod beanMethod) {
		// ...
		ConfigurationClassBeanDefinition beanDef = new ConfigurationClassBeanDefinition(configClass, metadata, beanName);

		BeanDefinition beanDefToRegister = beanDef;
		
		// ...
		this.registry.registerBeanDefinition(beanName, beanDefToRegister);
	}

	// 调用 registerBeanDefinitions 实现注册
	private void loadBeanDefinitionsFromRegistrars(Map<ImportBeanDefinitionRegistrar, AnnotationMetadata> registrars) {
		registrars.forEach((registrar, metadata) ->
				registrar.registerBeanDefinitions(metadata, this.registry, this.importBeanNameGenerator));
	}
}
```

## 参考资料

- [ConfigurationClassPostProcessor—Spring中最最最重要的后置处理器！没有之一！！！](https://www.cnblogs.com/lucky-yqy/p/14106107.html)
- [Spring源码解析 – @Configuration配置类及注解Bean的解析](https://www.cnblogs.com/ashleyboy/p/9667485.html)
- [@Configuration 注解使用及源码解析](https://www.cnblogs.com/zjdxr-up/p/14058667.html)
- [SpringBoot源码-@Configuration注解的解析](https://segmentfault.com/a/1190000020625414)


