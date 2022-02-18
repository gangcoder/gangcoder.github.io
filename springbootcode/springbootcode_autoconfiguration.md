<!-- ---
title: SpringBoot Code @EnableAutoConfiguration 自动配置分析
date: 2021-06-30 12:40:27
category: java100, springbootcode
--- -->

# SpringBoot Code @EnableAutoConfiguration 自动配置分析

## 1. 注解结构

EnableAutoConfiguration 重点是通过 Import 引入 AutoConfigurationImportSelector。

AutoConfigurationImportSelector 中实现自动配置类加载。

```java
@AutoConfigurationPackage
@Import(AutoConfigurationImportSelector.class)
public @interface EnableAutoConfiguration {
}
```

注册一个 BasePackages，用来保存扫描路径。

```java
@Import(AutoConfigurationPackages.Registrar.class)
public @interface AutoConfigurationPackage {

}
```

## 2. AutoConfigurationImportSelector 导入

AutoConfigurationImportSelector 属于延迟导入，实际调用 

```java
public class AutoConfigurationImportSelector implements DeferredImportSelector, BeanClassLoaderAware,
		ResourceLoaderAware, BeanFactoryAware, EnvironmentAware, Ordered {
	@Override
	public Class<? extends Group> getImportGroup() {
		return AutoConfigurationGroup.class;
	}
}
```

延迟导入 AutoConfigurationGroup 实现。

```java
static class AutoConfigurationGroup implements DeferredImportSelector.Group, BeanClassLoaderAware, BeanFactoryAware, ResourceLoaderAware {
    @Override
    public void process(AnnotationMetadata annotationMetadata, DeferredImportSelector deferredImportSelector) {
        // 调用 getAutoConfigurationEntry，加载所有需要自动配置的全类名
        AutoConfigurationEntry autoConfigurationEntry = ((AutoConfigurationImportSelector) deferredImportSelector)
                .getAutoConfigurationEntry(getAutoConfigurationMetadata(), annotationMetadata);

        // 暂存需要加载的全类名
        this.autoConfigurationEntries.add(autoConfigurationEntry);
        
        // ...
    }

    // 获取暂存的全类名
    @Override
    public Iterable<Entry> selectImports() {
        // ...

        // 暂存的全类名转为set
        Set<String> processedConfigurations = this.autoConfigurationEntries.stream()
                .map(AutoConfigurationEntry::getConfigurations).flatMap(Collection::stream)
                .collect(Collectors.toCollection(LinkedHashSet::new));
        
        // ...

        // 排序后返回全类名，之后会加入 configClass，最后加载为 bean 定义，并且注册到容器
        return sortAutoConfigurations(processedConfigurations, getAutoConfigurationMetadata()).stream()
                .map((importClassName) -> new Entry(this.entries.get(importClassName), importClassName))
                .collect(Collectors.toList());
    }
}
```

## 3. 加载自动配置类

从 `META-INF/spring.factories` 配置文件加载 `EnableAutoConfiguration.class` 自动配置全类名。

```java
public class AutoConfigurationImportSelector implements DeferredImportSelector, BeanClassLoaderAware,
		ResourceLoaderAware, BeanFactoryAware, EnvironmentAware, Ordered {

	protected AutoConfigurationEntry getAutoConfigurationEntry(AutoConfigurationMetadata autoConfigurationMetadata,
			AnnotationMetadata annotationMetadata) {
		// ...
        // 加载自动配置全类名
		List<String> configurations = getCandidateConfigurations(annotationMetadata, attributes);

        // 自动配置类过滤，根据 OnCondition 条件过滤
		configurations = filter(configurations, autoConfigurationMetadata);
		
        // 返回过滤后可以加载的全类名
		return new AutoConfigurationEntry(configurations, exclusions);
	}

	protected List<String> getCandidateConfigurations(AnnotationMetadata metadata, AnnotationAttributes attributes) {
        // 加载 EnableAutoConfiguration.class 配置项
		List<String> configurations = SpringFactoriesLoader.loadFactoryNames(getSpringFactoriesLoaderFactoryClass(),
				getBeanClassLoader());
		// ...
		return configurations;
	}

	private List<String> filter(List<String> configurations, AutoConfigurationMetadata autoConfigurationMetadata) {
		// ...
		String[] candidates = StringUtils.toStringArray(configurations);
		
        // 加载过滤器
		for (AutoConfigurationImportFilter filter : getAutoConfigurationImportFilters()) {
			// 过滤是否满足条件
			boolean[] match = filter.match(candidates, autoConfigurationMetadata);
			for (int i = 0; i < match.length; i++) {
				if (!match[i]) {
					skip[i] = true;
				}
			}
		}
		
        // ...

		List<String> result = new ArrayList<>(candidates.length);
		for (int i = 0; i < candidates.length; i++) {
			if (!skip[i]) {
				result.add(candidates[i]);
			}
		}
		
        // ...
		return new ArrayList<>(result);
	}

	protected List<AutoConfigurationImportFilter> getAutoConfigurationImportFilters() {
		return SpringFactoriesLoader.loadFactories(AutoConfigurationImportFilter.class, this.beanClassLoader);
	}

}
```

## 参考资料

- [Spring Boot @EnableAutoConfiguration注解分析](https://www.cnblogs.com/ashleyboy/p/9562052.html)
- [SpringBoot 源码解析 （五）----- Spring Boot的核心能力 - 自动配置源码解析](https://www.cnblogs.com/java-chen-hao/p/11837043.html)
