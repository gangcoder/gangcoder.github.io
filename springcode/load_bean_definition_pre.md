# Load Bean Definition
## 1. Bean 定义文件加载
```java
loadBeanDefinitions (DefaultListableBeanFactory beanFactory)
```

```java
XmlBeanDefinitionReader beanDefinitionReader = new XmlBeanDefinitionReader(beanFactory);
initBeanDefinitionReader(beanDefinitionReader);
```

```java
loadBeanDefinitions(beanDefinitionReader)
```

```java
void loadBeanDefinitions (XmlBeanDefinitionReader reader)
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
Resource[] resources = ((ResourcePatternResolver) resourceLoader).getResources(location);
```

```java
int count = loadBeanDefinitions(resources);
```

```java
int loadBeanDefinitions (EncodedResource encodedResource)
```

```java
InputSource inputSource = new InputSource(inputStream);
return doLoadBeanDefinitions(inputSource, encodedResource.getResource());
```

```java
int doLoadBeanDefinitions (InputSource inputSource, Resource resource)
```

```java
Document doc = doLoadDocument(inputSource, resource);
int count = registerBeanDefinitions(doc, resource);
return count;
```

```java
int registerBeanDefinitions (Document doc, Resource resource)
```

```java
BeanDefinitionDocumentReader documentReader = createBeanDefinitionDocumentReader();
```

```java
documentReader.registerBeanDefinitions(doc, createReaderContext(resource));
```

## 2. 加载 BeanDefinition
```java
registerBeanDefinitions (Document doc, XmlReaderContext readerContext)
```

```java
doRegisterBeanDefinitions(doc.getDocumentElement());
```

```java
doRegisterBeanDefinitions (Element root)
```

```java
this.delegate = createDelegate(getReaderContext(), root, parent);
preProcessXml(root);
```

```java
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
NodeList nl = root.getChildNodes();
for (int i = 0; i < nl.getLength(); i++) {
```

```java
parseDefaultElement(ele, delegate);
```

```java
parseDefaultElement (Element ele, BeanDefinitionParserDelegate delegate)
```

```java
processBeanDefinition(ele, delegate);
```

```java
processBeanDefinition (Element ele, BeanDefinitionParserDelegate delegate)
```

```java
BeanDefinitionHolder bdHolder = delegate.parseBeanDefinitionElement(ele);
```

```java
BeanDefinitionReaderUtils.registerBeanDefinition(bdHolder, getReaderContext().getRegistry());
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
String beanName = id;
AbstractBeanDefinition beanDefinition = parseBeanDefinitionElement(ele, beanName, containingBean);
return new BeanDefinitionHolder(beanDefinition, beanName, aliasesArray);
```

```java
AbstractBeanDefinition parseBeanDefinitionElement (Element ele, String beanName, @Nullable BeanDefinition containingBean)
```

```java
className = ele.getAttribute(CLASS_ATTRIBUTE).trim();
AbstractBeanDefinition bd = createBeanDefinition(className, parent);
```

```java
parseBeanDefinitionAttributes(ele, beanName, containingBean, bd);
parseConstructorArgElements(ele, bd);
parsePropertyElements(ele, bd);
bd.setSource(extractSource(ele));
return bd;
```

## 4. BeanDefinition 注册到 BeanFactory
```java
registerBeanDefinition (BeanDefinitionHolder definitionHolder, BeanDefinitionRegistry registry)
```

```java
String beanName = definitionHolder.getBeanName();
registry.registerBeanDefinition(beanName, definitionHolder.getBeanDefinition());
```

```java
registerBeanDefinition (String beanName, BeanDefinition beanDefinition)
```

```java
this.beanDefinitionMap.put(beanName, beanDefinition);
this.beanDefinitionNames.add(beanName);
```
