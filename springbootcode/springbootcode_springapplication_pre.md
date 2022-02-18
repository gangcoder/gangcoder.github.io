SpringApplication
+ SpringApplication (ResourceLoader resourceLoader, Class<?>... primarySources)
- Collection<T> getSpringFactoriesInstances (Class<T> type)
- Collection<T> getSpringFactoriesInstances (Class<T> type, Class<?>[] parameterTypes, Object... args)
- List<T> createSpringFactoriesInstances (Class<T> type, Class<?>[] parameterTypes, ClassLoader classLoader, ...)
+ ConfigurableApplicationContext run (String... args)
- SpringApplicationRunListeners getRunListeners (String[] args)
- ConfigurableEnvironment prepareEnvironment (SpringApplicationRunListeners listeners, ApplicationArguments applicationArguments)
- void prepareContext (ConfigurableApplicationContext context, ConfigurableEnvironment environment, SpringApplicationRunListeners listeners, ...)
- void applyInitializers (ConfigurableApplicationContext context)
+ Set<Object> getAllSources ()
- void load (ApplicationContext context, Object[] sources)
- void callRunners (ApplicationContext context, ApplicationArguments args)
- void callRunner (ApplicationRunner runner, ApplicationArguments args)

SpringFactoriesLoader
+ List<String> loadFactoryNames (Class<?> factoryType, @Nullable ClassLoader classLoader)
- Map<String, List<String>> loadSpringFactories (@Nullable ClassLoader classLoader)

SpringApplicationRunListeners
- final List<SpringApplicationRunListener> listeners
--
+ SpringApplicationRunListeners (Log log, Collection<? extends SpringApplicationRunListener> listeners)
- void starting ()

EventPublishingRunListener
+ void starting ()

SimpleApplicationEventMulticaster
+ void multicastEvent (final ApplicationEvent event, @Nullable ResolvableType eventType)

BeanDefinitionLoader
- int load (Class<?> source)
- boolean isComponent (Class<?> type)


# SpringApplication 初始化过程

## 1. SpringApplication 类

```java
SpringApplication (ResourceLoader resourceLoader, Class<?>... primarySources)
```

```java
this.primarySources = new LinkedHashSet<>(Arrays.asList(primarySources));
this.webApplicationType = WebApplicationType.deduceFromClasspath();
```

```java
setInitializers((Collection) getSpringFactoriesInstances(ApplicationContextInitializer.class));
```

```java
setListeners((Collection) getSpringFactoriesInstances(ApplicationListener.class));
this.mainApplicationClass = deduceMainApplicationClass();
```

### 1.1 设置初始化器 Initializer

```java
Collection<T> getSpringFactoriesInstances (Class<T> type)
```

```java
return getSpringFactoriesInstances(type, new Class<?>[] {});
```

```java
Collection<T> getSpringFactoriesInstances (Class<T> type, Class<?>[] parameterTypes, Object... args)
```

```java
ClassLoader classLoader = getClassLoader();
Set<String> names = new LinkedHashSet<>(SpringFactoriesLoader.loadFactoryNames(type, classLoader));
```

```java
List<T> instances = createSpringFactoriesInstances(type, parameterTypes, classLoader, args, names);
AnnotationAwareOrderComparator.sort(instances);
return instances;
```

```java
List<T> createSpringFactoriesInstances (Class<T> type, Class<?>[] parameterTypes, ClassLoader classLoader, ...)
```

```java
for (String name : names) {
    Class<?> instanceClass = ClassUtils.forName(name, classLoader);
    Constructor<?> constructor = instanceClass.getDeclaredConstructor(parameterTypes);
    T instance = (T) BeanUtils.instantiateClass(constructor, args);
    instances.add(instance);
}
return instances;
```

```java
List<String> loadFactoryNames (Class<?> factoryType, @Nullable ClassLoader classLoader)
```

```java
String factoryTypeName = factoryType.getName();
return loadSpringFactories(classLoader).getOrDefault(factoryTypeName, Collections.emptyList());
```

```java
Map<String, List<String>> loadSpringFactories (@Nullable ClassLoader classLoader)
```

```java
Enumeration<URL> urls = classLoader.getResources(FACTORIES_RESOURCE_LOCATION) ;
result = new LinkedMultiValueMap<>();
```

```java
while (urls.hasMoreElements()) {
    URL url = urls.nextElement();
    UrlResource resource = new UrlResource(url);
    Properties properties = PropertiesLoaderUtils.loadProperties(resource);
    for (Map.Entry<?, ?> entry : properties.entrySet()) {
        String factoryTypeName = ((String) entry.getKey()).trim();
        for (String factoryImplementationName : StringUtils.commaDelimitedListToStringArray((String) entry.getValue())) {
            result.add(factoryTypeName, factoryImplementationName.trim());
        }
    }
}
return result;
```

## 2. run 方法
```java
ConfigurableApplicationContext run (String... args)
```

```java
SpringApplicationRunListeners listeners = getRunListeners(args);
listeners.starting();
```

```java
ConfigurableEnvironment environment = prepareEnvironment(listeners, applicationArguments);
configureIgnoreBeanInfo(environment);
```

```java
context = createApplicationContext();
```

```java
prepareContext(context, environment, listeners, applicationArguments, printedBanner);
```

```java
refreshContext(context);
```

```java
callRunners(context, applicationArguments);
```

### 2.1 获取并启动监听器
```java
SpringApplicationRunListeners getRunListeners (String[] args)
```

```java
Class<?>[] types = new Class<?>[] { SpringApplication.class, String[].class };
return new SpringApplicationRunListeners(logger, getSpringFactoriesInstances(SpringApplicationRunListener.class, types, this, args));
```

```java
SpringApplicationRunListeners (Log log, Collection<? extends SpringApplicationRunListener> listeners)
```

```java
this.log = log;
this.listeners = new ArrayList<>(listeners);
```

```java
void starting ()
```

```java
for (SpringApplicationRunListener listener : this.listeners) {
    listener.starting();
}
```

```java
void starting ()
```

```java
this.initialMulticaster.multicastEvent(new ApplicationStartingEvent(this.application, this.args));
```

```java
void multicastEvent (final ApplicationEvent event, @Nullable ResolvableType eventType)
```

```java
for (ApplicationListener<?> listener : getApplicationListeners(event, type)) {
    invokeListener(listener, event);
}
```

### 2.2 准备 environment
```java
ConfigurableEnvironment prepareEnvironment (SpringApplicationRunListeners listeners, ApplicationArguments applicationArguments)
```

```java
ConfigurableEnvironment environment = getOrCreateEnvironment();
```

```java
listeners.environmentPrepared(environment);
return environment;
```

### 2.3 Spring 容器前置处理

```java
void prepareContext (ConfigurableApplicationContext context, ConfigurableEnvironment environment, SpringApplicationRunListeners listeners, ...)
```

```java
applyInitializers(context);
```

```java
Set<Object> sources = getAllSources();
```

```java
load(context, sources.toArray(new Object[0]));
listeners.contextLoaded(context);
```

```java
void applyInitializers (ConfigurableApplicationContext context)
```

```java
for (ApplicationContextInitializer initializer : getInitializers()) {
    initializer.initialize(context);
}
```

```java
Set<Object> getAllSources ()
```

```java
Set<Object> allSources = new LinkedHashSet<>();
allSources.addAll(this.primarySources);
```

```java
void load (ApplicationContext context, Object[] sources)
```

```java
BeanDefinitionLoader loader = createBeanDefinitionLoader(getBeanDefinitionRegistry(context), sources);
loader.load();
```

```java
int load (Class<?> source)
```

```java
if (isComponent(source)) {
    this.annotatedReader.register(source);
    return 1;
}
```

```java
boolean isComponent (Class<?> type)
```

```java
if (MergedAnnotations.from(type, SearchStrategy.TYPE_HIERARCHY).isPresent(Component.class)) {
    return true;
}
```

### 2.4 执行Runners
```java
void callRunners (ApplicationContext context, ApplicationArguments args)
```

```java
List<Object> runners = new ArrayList<>();
runners.addAll(context.getBeansOfType(ApplicationRunner.class).values());
```

```java
runners.addAll(context.getBeansOfType(CommandLineRunner.class).values());
```

```java
AnnotationAwareOrderComparator.sort(runners);
for (Object runner : new LinkedHashSet<>(runners)) {
    if (runner instanceof ApplicationRunner) {
        callRunner((ApplicationRunner) runner, args);
    }
}
```

```java
void callRunner (ApplicationRunner runner, ApplicationArguments args)
```

```java
(runner).run(args);
```
