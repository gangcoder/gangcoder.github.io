AbstractApplicationContext
- prepareBeanFactory (ConfigurableListableBeanFactory beanFactory)
- invokeBeanFactoryPostProcessors (ConfigurableListableBeanFactory beanFactory)
- registerBeanPostProcessors (ConfigurableListableBeanFactory beanFactory)
- initApplicationEventMulticaster ()
- registerListeners ()
- finishRefresh ()
+ publishEvent (ApplicationEvent event)
- publishEvent (Object event, @Nullable ResolvableType eventType)

PostProcessorRegistrationDelegate
+ invokeBeanFactoryPostProcessors (ConfigurableListableBeanFactory beanFactory, List<BeanFactoryPostProcessor> beanFactoryPostProcessors)
- invokeBeanFactoryPostProcessors (Collection<? extends BeanFactoryPostProcessor> postProcessors, ConfigurableListableBeanFactory beanFactory)
+ registerBeanPostProcessors (ConfigurableListableBeanFactory beanFactory, AbstractApplicationContext applicationContext)
- registerBeanPostProcessors (ConfigurableListableBeanFactory beanFactory, List<BeanPostProcessor> postProcessors)


# refresh 方法解析
## 1.  prepareBeanFactory
```java
prepareBeanFactory (ConfigurableListableBeanFactory beanFactory)
```

```java
beanFactory.setBeanClassLoader(getClassLoader());
beanFactory.setBeanExpressionResolver(new StandardBeanExpressionResolver(beanFactory.getBeanClassLoader()));
beanFactory.addPropertyEditorRegistrar(new ResourceEditorRegistrar(this, getEnvironment()));
```

```java
beanFactory.registerSingleton(ENVIRONMENT_BEAN_NAME, getEnvironment());
beanFactory.registerSingleton(SYSTEM_PROPERTIES_BEAN_NAME, getEnvironment().getSystemProperties());
beanFactory.registerSingleton(SYSTEM_ENVIRONMENT_BEAN_NAME, getEnvironment().getSystemEnvironment());
```

## 2.  invokeBeanFactoryPostProcessors
```java
invokeBeanFactoryPostProcessors (ConfigurableListableBeanFactory beanFactory)
```

```java
PostProcessorRegistrationDelegate.invokeBeanFactoryPostProcessors(beanFactory, getBeanFactoryPostProcessors());
```

```java
invokeBeanFactoryPostProcessors (ConfigurableListableBeanFactory beanFactory, List<BeanFactoryPostProcessor> beanFactoryPostProcessors)
```

```java
String[] postProcessorNames = beanFactory.getBeanNamesForType(BeanFactoryPostProcessor.class, true, false);
```

```java
for (String ppName : postProcessorNames) {
    if (processedBeans.contains(ppName)) {
    } else if (beanFactory.isTypeMatch(ppName, Ordered.class)) {
        orderedPostProcessorNames.add(ppName);
    }
}
```

```java
List<BeanFactoryPostProcessor> orderedPostProcessors = new ArrayList<>(orderedPostProcessorNames.size());
for (String postProcessorName : orderedPostProcessorNames) {
    orderedPostProcessors.add(beanFactory.getBean(postProcessorName, BeanFactoryPostProcessor.class));
}
sortPostProcessors(orderedPostProcessors, beanFactory);
```

```java
invokeBeanFactoryPostProcessors(orderedPostProcessors, beanFactory);
```

```java
invokeBeanFactoryPostProcessors (Collection<? extends BeanFactoryPostProcessor> postProcessors, ConfigurableListableBeanFactory beanFactory)
```

```java
for (BeanFactoryPostProcessor postProcessor : postProcessors) {
    postProcessor.postProcessBeanFactory(beanFactory);
}
```

## 3.  registerBeanPostProcessors
```java
registerBeanPostProcessors (ConfigurableListableBeanFactory beanFactory)
```

```java
PostProcessorRegistrationDelegate.registerBeanPostProcessors(beanFactory, this);
```

```java
registerBeanPostProcessors (ConfigurableListableBeanFactory beanFactory, AbstractApplicationContext applicationContext)
```

```java
String[] postProcessorNames = beanFactory.getBeanNamesForType(BeanPostProcessor.class, true, false);
```

```java
for (String ppName : postProcessorNames) {
    if (beanFactory.isTypeMatch(ppName, PriorityOrdered.class)) {
    } else if (beanFactory.isTypeMatch(ppName, Ordered.class)) {
        orderedPostProcessorNames.add(ppName);
    }
}
```

```java
List<BeanPostProcessor> orderedPostProcessors = new ArrayList<>(orderedPostProcessorNames.size());
for (String ppName : orderedPostProcessorNames) {
    BeanPostProcessor pp = beanFactory.getBean(ppName, BeanPostProcessor.class);
    orderedPostProcessors.add(pp);
}
sortPostProcessors(orderedPostProcessors, beanFactory);
```

```java
registerBeanPostProcessors(beanFactory, orderedPostProcessors);
```

```java
registerBeanPostProcessors (ConfigurableListableBeanFactory beanFactory, List<BeanPostProcessor> postProcessors)
```

```java
for (BeanPostProcessor postProcessor : postProcessors) {
    beanFactory.addBeanPostProcessor(postProcessor);
}
```

## 4.  initApplicationEventMulticaster
```java
initApplicationEventMulticaster ()
```

```java
this.applicationEventMulticaster = new SimpleApplicationEventMulticaster(beanFactory);
beanFactory.registerSingleton(APPLICATION_EVENT_MULTICASTER_BEAN_NAME, this.applicationEventMulticaster);
}
```

## 5.  registerListeners
```java
registerListeners ()
```

```java
for (ApplicationListener<?> listener : getApplicationListeners()) {
    getApplicationEventMulticaster().addApplicationListener(listener);
}
```

```java
String[] listenerBeanNames = getBeanNamesForType(ApplicationListener.class, true, false);
for (String listenerBeanName : listenerBeanNames) {
    getApplicationEventMulticaster().addApplicationListenerBean(listenerBeanName);
}
```

## 6.  finishRefresh
```java
finishRefresh ()
```

```java
initLifecycleProcessor();
getLifecycleProcessor().onRefresh();
```

```java
publishEvent(new ContextRefreshedEvent(this));
```

```java
publishEvent (ApplicationEvent event)
```

```java
publishEvent(event, null);
```

```java
publishEvent (Object event, @Nullable ResolvableType eventType)
```

```java
applicationEvent = (ApplicationEvent) event;
```

```java
getApplicationEventMulticaster().multicastEvent(applicationEvent, eventType);
}
```
