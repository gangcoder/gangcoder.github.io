PropertyResourceConfigurer
+ void postProcessBeanFactory (ConfigurableListableBeanFactory beanFactory)

PropertiesLoaderSupport
- Properties mergeProperties ()
- void loadProperties (Properties props)

PropertiesLoaderUtils
- void fillProperties (Properties props, EncodedResource resource, PropertiesPersister persister)

PropertyPlaceholderConfigurer
- void processProperties (ConfigurableListableBeanFactory beanFactoryToProcess, Properties props)
- void doProcessProperties (ConfigurableListableBeanFactory beanFactoryToProcess, StringValueResolver valueResolver)

BeanDefinitionVisitor
+ void visitBeanDefinition (BeanDefinition beanDefinition)
- void visitPropertyValues (MutablePropertyValues pvs)
- Object resolveValue (@Nullable Object value)
- String resolveStringValue (String strVal)

PlaceholderResolvingStringValueResolver
+ String resolveStringValue (String strVal)



public abstract class PropertyResourceConfigurer {
    public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException;
}

public abstract class PropertiesLoaderSupport {
    protected Properties mergeProperties() throws IOException;
    protected void loadProperties(Properties props) throws IOException;
}

public abstract class PropertiesLoaderUtils {
     static void fillProperties(Properties props, EncodedResource resource, PropertiesPersister persister) throws IOException;
}

public class PropertyPlaceholderConfigurer {
    protected void processProperties(ConfigurableListableBeanFactory beanFactoryToProcess, Properties props) throws BeansException;
    protected void doProcessProperties(ConfigurableListableBeanFactory beanFactoryToProcess, StringValueResolver valueResolver);
}

public class BeanDefinitionVisitor {
    public void visitBeanDefinition(BeanDefinition beanDefinition);
    protected void visitPropertyValues(MutablePropertyValues pvs);
    protected Object resolveValue(@Nullable Object value);
    protected String resolveStringValue(String strVal);
}

 class PlaceholderResolvingStringValueResolver {
    public String resolveStringValue(String strVal) throws BeansException;
}

# 占位符替换实现

## 1. 属性替换调用
```java
void postProcessBeanFactory (ConfigurableListableBeanFactory beanFactory)
```

```java
Properties mergedProps = mergeProperties();
convertProperties(mergedProps);
```

```java
processProperties(beanFactory, mergedProps);
```

## 2. properties 文件读取

```java
Properties mergeProperties ()
```

```java
loadProperties(result);
return result;
```

```java
void loadProperties (Properties props)
```

```java
for (Resource location : this.locations) {
    PropertiesLoaderUtils.fillProperties(props, new EncodedResource(location, this.fileEncoding), this.propertiesPersister);
}
```

```java
void fillProperties (Properties props, EncodedResource resource, PropertiesPersister persister)
```

```java
stream = resource.getInputStream();
persister.load(props, stream);
```

## 3. 替换实现
```java
void processProperties (ConfigurableListableBeanFactory beanFactoryToProcess, Properties props)
```

```java
StringValueResolver valueResolver = new PlaceholderResolvingStringValueResolver(props);
doProcessProperties(beanFactoryToProcess, valueResolver);
```

```java
void doProcessProperties (ConfigurableListableBeanFactory beanFactoryToProcess, StringValueResolver valueResolver)
```

```java
BeanDefinitionVisitor visitor = new BeanDefinitionVisitor(valueResolver);
String[] beanNames = beanFactoryToProcess.getBeanDefinitionNames();
```

```java
for (String curName : beanNames) {
    BeanDefinition bd = beanFactoryToProcess.getBeanDefinition(curName);
    visitor.visitBeanDefinition(bd);
}
```

```java
void visitBeanDefinition (BeanDefinition beanDefinition)
```

```java
visitParentName(beanDefinition);
if (beanDefinition.hasPropertyValues()) {
    visitPropertyValues(beanDefinition.getPropertyValues());
}
```

```java
void visitPropertyValues (MutablePropertyValues pvs)
```

```java
PropertyValue[] pvArray = pvs.getPropertyValues();
for (PropertyValue pv : pvArray) {
    Object newVal = resolveValue(pv.getValue());
    pvs.add(pv.getName(), newVal);
}
```

```java
Object resolveValue (@Nullable Object value)
```

```java
String visitedString = resolveStringValue(stringValue);
typedStringValue.setValue(visitedString);
```

```java
String resolveStringValue (String strVal)
```

```java
String resolvedValue = this.valueResolver.resolveStringValue(strVal);
return (strVal.equals(resolvedValue) ? strVal : resolvedValue);
```

```java
String resolveStringValue (String strVal)
```

```java
String resolved = this.helper.replacePlaceholders(strVal, this.resolver);
return (resolved.equals(nullValue) ? null : resolved);
```
