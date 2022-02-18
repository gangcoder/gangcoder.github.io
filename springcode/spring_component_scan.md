<!-- ---
title: 包扫描标签component-scan 解析
date: 2022-01-29 07:20:20
category: java100, spring, springcode
--- -->

# 包扫描标签component-scan 解析

![](draw/spring_component_scan.svg)

## 1. 注册解析器

解析到自定义标签时，根据namespace 找到对应处理器，并且初始化，初始化时注册具体标签解析器。

```java
public class BeanDefinitionParserDelegate {
    @Nullable
	public BeanDefinition parseCustomElement(Element ele, @Nullable BeanDefinition containingBd) {
        // 找到 namespace
		String namespaceUri = getNamespaceURI(ele);
		// ...

        // 找到处理器，内部会初始化
		NamespaceHandler handler = this.readerContext.getNamespaceHandlerResolver().resolve(namespaceUri);
		
        // ...
        // 调用解析处理
		return handler.parse(ele, new ParserContext(this.readerContext, this, containingBd));
	}
}

public class DefaultNamespaceHandlerResolver implements NamespaceHandlerResolver {
	@Override
	@Nullable
	public NamespaceHandler resolve(String namespaceUri) {
        // ...
        NamespaceHandler namespaceHandler = (NamespaceHandler) BeanUtils.instantiateClass(handlerClass);
        namespaceHandler.init();
        handlerMappings.put(namespaceUri, namespaceHandler);

        // ...
        return namespaceHandler;
    }
}
```

context 对应 ContextNamespaceHandler，这里进行初始化。

```java
public class ContextNamespaceHandler extends NamespaceHandlerSupport {
    public void init() {
        registerBeanDefinitionParser("component-scan", new ComponentScanBeanDefinitionParser());
    }
}

public abstract class NamespaceHandlerSupport implements NamespaceHandler {
    private final Map<String, BeanDefinitionParser> parsers = new HashMap<>();

	protected final void registerBeanDefinitionParser(String elementName, BeanDefinitionParser parser) {
		this.parsers.put(elementName, parser);
	}
}
```

## 2. 自定义标签解析

遇到自定义标签时，找到对应解析器处理。

```java
public abstract class NamespaceHandlerSupport implements NamespaceHandler {
	@Override
	@Nullable
	public BeanDefinition parse(Element element, ParserContext parserContext) {
		BeanDefinitionParser parser = findParserForElement(element, parserContext);
		return (parser != null ? parser.parse(element, parserContext) : null);
	}
}

public class ComponentScanBeanDefinitionParser implements BeanDefinitionParser {
	@Override
	@Nullable
	public BeanDefinition parse(Element element, ParserContext parserContext) {
        // 获取扫描范围
		String basePackage = element.getAttribute(BASE_PACKAGE_ATTRIBUTE);
		basePackage = parserContext.getReaderContext().getEnvironment().resolvePlaceholders(basePackage);
		String[] basePackages = StringUtils.tokenizeToStringArray(basePackage,
				ConfigurableApplicationContext.CONFIG_LOCATION_DELIMITERS);

		// 配置扫描器
		ClassPathBeanDefinitionScanner scanner = configureScanner(parserContext, element);

        // 扫描处理
		Set<BeanDefinitionHolder> beanDefinitions = scanner.doScan(basePackages);
		// ...

		return null;
	}

	protected ClassPathBeanDefinitionScanner configureScanner(ParserContext parserContext, Element element) {
		// ...
		ClassPathBeanDefinitionScanner scanner = createScanner(parserContext.getReaderContext(), useDefaultFilters);
		
        // 扫描器配置
		parseTypeFilters(element, scanner, parserContext);

		return scanner;
	}
}
```

## 3. 扫描逻辑实现

findCandidateComponents（basePackages）根据指定的扫描路径扫描并解析成beandefine, 后面通过 registerBeanDefinition(definitionHolder,this.regsistry) 将这些 beandefine 注册到IOC容器。


```java
public class ClassPathBeanDefinitionScanner extends ClassPathScanningCandidateComponentProvider {
	protected Set<BeanDefinitionHolder> doScan(String... basePackages) {
		// ...

		for (String basePackage : basePackages) {
            // 根据路径扫描，获取BeanDefinition
			// 默认Component 会被扫描
			Set<BeanDefinition> candidates = findCandidateComponents(basePackage);

			for (BeanDefinition candidate : candidates) {
				// 生成 beanName 
				String beanName = this.beanNameGenerator.generateBeanName(candidate, this.registry);
				
                // 注解配置信息读取，包括 Lazy, primary DependsOn, Role ,Description
				if (candidate instanceof AnnotatedBeanDefinition) {
					AnnotationConfigUtils.processCommonDefinitionAnnotations((AnnotatedBeanDefinition) candidate);
				}

                // ...
                // 检查beanName 是否注册
				if (checkCandidate(beanName, candidate)) {
					BeanDefinitionHolder definitionHolder = new BeanDefinitionHolder(candidate, beanName);
                    // ...
					// 注册BeanDefinition 到容器
					registerBeanDefinition(definitionHolder, this.registry);
				}
			}
		}
		return beanDefinitions;
	}

}

public class ClassPathScanningCandidateComponentProvider implements EnvironmentCapable, ResourceLoaderAware {
	public Set<BeanDefinition> findCandidateComponents(String basePackage) {
		if (this.componentsIndex != null && indexSupportsIncludeFilters()) {
			// ...
		}
		else {
			return scanCandidateComponents(basePackage);
		}
	}

	private Set<BeanDefinition> scanCandidateComponents(String basePackage) {
        // 找到路径下所有class 对象
        Resource[] resources = getResourcePatternResolver().getResources(packageSearchPath);

        for (Resource resource : resources) {
            // 将对象封装为 BeanDefinition
            MetadataReader metadataReader = getMetadataReaderFactory().getMetadataReader(resource);

            // 是否是需要注册的 bean 对象
            if (isCandidateComponent(metadataReader)) {
                ScannedGenericBeanDefinition sbd = new ScannedGenericBeanDefinition(metadataReader);
                sbd.setSource(resource);
                if (isCandidateComponent(sbd)) {
                    // ...
                    candidates.add(sbd);
                }
            }
        }

        return candidates;
    }
}
```


## 参考资料

- []()