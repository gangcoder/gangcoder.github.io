
# XML 应用上下文启动实现
## 创建应用
```java
ClassPathXmlApplicationContext (String configLocation)
```

```java
this(new String[] { configLocation }, true, null);
```

```java
ClassPathXmlApplicationContext (String[] configLocations, boolean refresh, ...)
```

```java
setConfigLocations(configLocations);
```

```java
if (refresh) {
    refresh();
}
```

## 应用刷新
```java
refresh ()
```

```java
prepareRefresh();
```

```java
ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();
```

```java
prepareBeanFactory(beanFactory);
```

```java
postProcessBeanFactory(beanFactory);
```

```java
invokeBeanFactoryPostProcessors(beanFactory);
```

```java
registerBeanPostProcessors(beanFactory);
```

```java
initMessageSource();
```

```java
initApplicationEventMulticaster();
```

```java
onRefresh();
```

```java
registerListeners();
```

```java
finishBeanFactoryInitialization(beanFactory);
```

```java
finishRefresh();
```

## 刷新前置处理
```java
prepareRefresh ()
```

```java
this.active.set(true);
initPropertySources();
```

## 创建 BeanFactory
```java
ConfigurableListableBeanFactory obtainFreshBeanFactory ()
```

```java
refreshBeanFactory();
return getBeanFactory();
```

```java
refreshBeanFactory ()
```

```java
DefaultListableBeanFactory beanFactory = createBeanFactory();
beanFactory.setSerializationId(getId());
customizeBeanFactory(beanFactory);
```

```java
loadBeanDefinitions(beanFactory);
this.beanFactory = beanFactory;
```

```java
DefaultListableBeanFactory createBeanFactory ()
```

```java
return new DefaultListableBeanFactory(getInternalParentBeanFactory());
```

```java
ConfigurableListableBeanFactory getBeanFactory ()
```

```java
DefaultListableBeanFactory beanFactory = this.beanFactory;
return beanFactory;
```

```java
loadBeanDefinitions (DefaultListableBeanFactory beanFactory)
```

```java
XmlBeanDefinitionReader beanDefinitionReader = new XmlBeanDefinitionReader(beanFactory);
beanDefinitionReader.setEnvironment(this.getEnvironment());
beanDefinitionReader.setResourceLoader(this);
beanDefinitionReader.setEntityResolver(new ResourceEntityResolver(this));
initBeanDefinitionReader(beanDefinitionReader);
```

```java
loadBeanDefinitions(beanDefinitionReader);
```
