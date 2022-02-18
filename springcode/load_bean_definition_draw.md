# Load Bean Definition
## 1. Bean 定义文件加载
```java
loadBeanDefinitions (DefaultListableBeanFactory beanFactory)
```

```java
XmlBeanDefinitionReader beanDefinitionReader = new XmlBeanDefinitionReader(beanFactory);
initBeanDefinitionReader(beanDefinitionReader);
loadBeanDefinitions(beanDefinitionReader);
```

```java
loadBeanDefinitions (XmlBeanDefinitionReader reader)
```

```java
String[] configLocations = getConfigLocations();
if (configLocations != null) {
    reader.loadBeanDefinitions(configLocations);
}
```

```java
int loadBeanDefinitions (String... locations)
```

```java
for (String location : locations) {
    count += loadBeanDefinitions(location);
}
```

```java
int loadBeanDefinitions (String location, @Nullable Set<Resource> actualResources)
```

```java
ResourceLoader resourceLoader = getResourceLoader();
if (resourceLoader instanceof ResourcePatternResolver) {
    try {
        Resource[] resources = ((ResourcePatternResolver) resourceLoader).getResources(location);
        int count = loadBeanDefinitions(resources);
    } catch (IOException ex) {
    }
}
```

```java
int loadBeanDefinitions (EncodedResource encodedResource)
```

```java
try (InputStream inputStream = encodedResource.getResource().getInputStream()) {
    InputSource inputSource = new InputSource(inputStream);
    return doLoadBeanDefinitions(inputSource, encodedResource.getResource());
} catch (IOException ex) {
}
```

```java
int doLoadBeanDefinitions (InputSource inputSource, Resource resource)
```

```java
try {
    Document doc = doLoadDocument(inputSource, resource);
    int count = registerBeanDefinitions(doc, resource);
    return count;
} catch (BeanDefinitionStoreException ex) {
    throw ex;
}
```

```java
int registerBeanDefinitions (Document doc, Resource resource)
```

```java
BeanDefinitionDocumentReader documentReader = createBeanDefinitionDocumentReader();
int countBefore = getRegistry().getBeanDefinitionCount();
documentReader.registerBeanDefinitions(doc, createReaderContext(resource));
return getRegistry().getBeanDefinitionCount() - countBefore;
```

## 2. 加载 BeanDefinition
```java
registerBeanDefinitions (Document doc, XmlReaderContext readerContext)
```

```java
this.readerContext = readerContext;
doRegisterBeanDefinitions(doc.getDocumentElement());
```

```java
doRegisterBeanDefinitions (Element root)
```

```java
this.delegate = createDelegate(getReaderContext(), root, parent);
preProcessXml(root);
parseBeanDefinitions(root, this.delegate);
postProcessXml(root);
this.delegate = parent;
```

```java
BeanDefinitionParserDelegate createDelegate (XmlReaderContext readerContext, Element root, @Nullable BeanDefinitionParserDelegate parentDelegate)
```

```java
BeanDefinitionParserDelegate delegate = new BeanDefinitionParserDelegate(readerContext);
delegate.initDefaults(root, parentDelegate);
return delegate;
```

```java
parseBeanDefinitions (Element root, BeanDefinitionParserDelegate delegate)
```

```java
if (delegate.isDefaultNamespace(root)) {
    NodeList nl = root.getChildNodes();
    for (int i = 0; i < nl.getLength(); i++) {
        Node node = nl.item(i);
        if (node instanceof Element) {
            Element ele = (Element) node;
            if (delegate.isDefaultNamespace(ele)) {
                parseDefaultElement(ele, delegate);
            } else {
                delegate.parseCustomElement(ele);
            }
        }
    }
} else {
    delegate.parseCustomElement(root);
}
```

```java
parseDefaultElement (Element ele, BeanDefinitionParserDelegate delegate)
```

```java
if (delegate.nodeNameEquals(ele, IMPORT_ELEMENT)) {
    importBeanDefinitionResource(ele);
} else if (delegate.nodeNameEquals(ele, BEAN_ELEMENT)) {
    processBeanDefinition(ele, delegate);
}
```

```java
processBeanDefinition (Element ele, BeanDefinitionParserDelegate delegate)
```

```java
BeanDefinitionHolder bdHolder = delegate.parseBeanDefinitionElement(ele);
if (bdHolder != null) {
    bdHolder = delegate.decorateBeanDefinitionIfRequired(ele, bdHolder);
    try {
        BeanDefinitionReaderUtils.registerBeanDefinition(bdHolder, getReaderContext().getRegistry());
    } catch (BeanDefinitionStoreException ex) {
        getReaderContext().error("Failed to register bean definition with name '" + bdHolder.getBeanName() + "'", ele, ex);
    }
}
```

## 3. BeanDefinition 解析
```java
BeanDefinitionHolder parseBeanDefinitionElement (Element ele)
```

```java
return parseBeanDefinitionElement(ele, null);
```

```java
BeanDefinitionHolder parseBeanDefinitionElement (Element ele, @Nullable BeanDefinition containingBean)
```

```java
String id = ele.getAttribute(ID_ATTRIBUTE);
String nameAttr = ele.getAttribute(NAME_ATTRIBUTE);
String beanName = id;
if (!StringUtils.hasText(beanName) && !aliases.isEmpty()) {
    beanName = aliases.remove(0);
    if (logger.isTraceEnabled()) {
        logger.trace("No XML 'id' specified - using '" + beanName + "' as bean name and " + aliases + " as aliases");
    }
}
AbstractBeanDefinition beanDefinition = parseBeanDefinitionElement(ele, beanName, containingBean);
if (beanDefinition != null) {
    String[] aliasesArray = StringUtils.toStringArray(aliases);
    return new BeanDefinitionHolder(beanDefinition, beanName, aliasesArray);
}
return null;
```

```java
AbstractBeanDefinition parseBeanDefinitionElement (Element ele, String beanName, @Nullable BeanDefinition containingBean)
```

```java
this.parseState.push(new BeanEntry(beanName));
String className = null;
if (ele.hasAttribute(CLASS_ATTRIBUTE)) {
    className = ele.getAttribute(CLASS_ATTRIBUTE).trim();
}
try {
    AbstractBeanDefinition bd = createBeanDefinition(className, parent);
    parseBeanDefinitionAttributes(ele, beanName, containingBean, bd);
    bd.setDescription(DomUtils.getChildElementValueByTagName(ele, DESCRIPTION_ELEMENT));
    parseMetaElements(ele, bd);
    parseLookupOverrideSubElements(ele, bd.getMethodOverrides());
    parseReplacedMethodSubElements(ele, bd.getMethodOverrides());
    parseConstructorArgElements(ele, bd);
    parsePropertyElements(ele, bd);
    parseQualifierElements(ele, bd);
    bd.setResource(this.readerContext.getResource());
    bd.setSource(extractSource(ele));
    return bd;
} catch (ClassNotFoundException ex) {
    error("Bean class [" + className + "] not found", ele, ex);
}
return null;
```

## 4. BeanDefinition 注册到 BeanFactory
```java
registerBeanDefinition (BeanDefinitionHolder definitionHolder, BeanDefinitionRegistry registry)
```

```java
String beanName = definitionHolder.getBeanName();
registry.registerBeanDefinition(beanName, definitionHolder.getBeanDefinition());
```

