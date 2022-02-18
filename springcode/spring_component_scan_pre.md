BeanDefinitionParserDelegate
+ BeanDefinition parseCustomElement (Element ele, @Nullable BeanDefinition containingBd)

DefaultNamespaceHandlerResolver
+ NamespaceHandler resolve (String namespaceUri)

ContextNamespaceHandler
+ void init ()

NamespaceHandlerSupport
- final Map<String, BeanDefinitionParser> parsers = new HashMap<>()
--
- void registerBeanDefinitionParser (String elementName, BeanDefinitionParser parser)
+ BeanDefinition parse (Element element, ParserContext parserContext)

ComponentScanBeanDefinitionParser
+ BeanDefinition parse (Element element, ParserContext parserContext)
- ClassPathBeanDefinitionScanner configureScanner (ParserContext parserContext, Element element)

ClassPathBeanDefinitionScanner
- Set<BeanDefinitionHolder> doScan (String... basePackages)

ClassPathScanningCandidateComponentProvider
+ Set<BeanDefinition> findCandidateComponents (String basePackage)
- Set<BeanDefinition> scanCandidateComponents (String basePackage)


# 包扫描注解 component-scan
## 1. 注册解析器
```java
BeanDefinition parseCustomElement (Element ele, @Nullable BeanDefinition containingBd)
```

```java
String namespaceUri = getNamespaceURI(ele);
NamespaceHandler handler = this.readerContext.getNamespaceHandlerResolver().resolve(namespaceUri);
```

```java
return handler.parse(ele, new ParserContext(this.readerContext, this, containingBd));
```

```java
NamespaceHandler resolve (String namespaceUri)
```

```java
NamespaceHandler namespaceHandler = (NamespaceHandler) BeanUtils.instantiateClass(handlerClass);
namespaceHandler.init();
```

```java
return namespaceHandler;
```

```java
void init ()
```

```java
registerBeanDefinitionParser("component-scan", new ComponentScanBeanDefinitionParser());
```

```java
void registerBeanDefinitionParser (String elementName, BeanDefinitionParser parser)
```

```java
this.parsers.put(elementName, parser);
```

## 2. 自定义标签解析
```java
BeanDefinition parse (Element element, ParserContext parserContext)
```

```java
BeanDefinitionParser parser = findParserForElement(element, parserContext);
return (parser != null ? parser.parse(element, parserContext) : null);
```

```java
BeanDefinition parse (Element element, ParserContext parserContext)
```

```java
String basePackage = element.getAttribute(BASE_PACKAGE_ATTRIBUTE);
basePackage = parserContext.getReaderContext().getEnvironment().resolvePlaceholders(basePackage);
```

```java
ClassPathBeanDefinitionScanner scanner = configureScanner(parserContext, element);
```

```java
Set<BeanDefinitionHolder> beanDefinitions = scanner.doScan(basePackages);
return null;
```

```java
ClassPathBeanDefinitionScanner configureScanner (ParserContext parserContext, Element element)
```

```java
ClassPathBeanDefinitionScanner scanner = createScanner(parserContext.getReaderContext(), useDefaultFilters);
parseTypeFilters(element, scanner, parserContext);
return scanner;
```

## 3. 扫描逻辑实现
```java
Set<BeanDefinitionHolder> doScan (String... basePackages)
```

```java
for (String basePackage : basePackages) {
    Set<BeanDefinition> candidates = findCandidateComponents(basePackage);
}
```

```java
for (BeanDefinition candidate : candidates) {
    String beanName = this.beanNameGenerator.generateBeanName(candidate, this.registry);
    BeanDefinitionHolder definitionHolder = new BeanDefinitionHolder(candidate, beanName);
    registerBeanDefinition(definitionHolder, this.registry);
}
```

```java
Set<BeanDefinition> findCandidateComponents (String basePackage)
```

```java
return scanCandidateComponents(basePackage);
```

```java
Set<BeanDefinition> scanCandidateComponents (String basePackage)
```

```java
Resource[] resources = getResourcePatternResolver().getResources(packageSearchPath);
```

```java
for (Resource resource : resources) {
    MetadataReader metadataReader = getMetadataReaderFactory().getMetadataReader(resource);
    ScannedGenericBeanDefinition sbd = new ScannedGenericBeanDefinition(metadataReader);
    candidates.add(sbd);
}
return candidates;
```
