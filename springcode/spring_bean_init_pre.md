AbstractApplicationContext
- void finishBeanFactoryInitialization (ConfigurableListableBeanFactory beanFactory)

DefaultListableBeanFactory
+ void preInstantiateSingletons ()

AbstractBeanFactory
+ Object getBean (String name)
- T doGetBean (String name, @Nullable Class<T> requiredType, @Nullable Object[] args, ...)

AbstractAutowireCapableBeanFactory
- Object createBean (String beanName, RootBeanDefinition mbd, @Nullable Object[] args)
- Object doCreateBean (String beanName, RootBeanDefinition mbd, @Nullable Object[] args)
- BeanWrapper createBeanInstance (String beanName, RootBeanDefinition mbd, @Nullable Object[] args)
- BeanWrapper instantiateBean (String beanName, RootBeanDefinition mbd)

SimpleInstantiationStrategy
+ Object instantiate (RootBeanDefinition bd, @Nullable String beanName, BeanFactory owner)

BeanUtils
+ T instantiateClass (Constructor<T> ctor, Object... args)

# Bean 实例化过程

## 1. preInstantiateSingletons 方法
```java
void finishBeanFactoryInitialization (ConfigurableListableBeanFactory beanFactory)
```

```java
beanFactory.preInstantiateSingletons();
```

```java
void preInstantiateSingletons ()
```

```java
List<String> beanNames = new ArrayList<>(this.beanDefinitionNames);
for (String beanName : beanNames) {
RootBeanDefinition bd = getMergedLocalBeanDefinition(beanName);
```

```java
getBean(beanName);
```

## 2. doGetBean 方法构造 Bean
```java
Object getBean (String name)
```

```java
return doGetBean(name, null, null, false);
```

```java
T doGetBean (String name, @Nullable Class<T> requiredType, @Nullable Object[] args, ...)
```

```java
String[] dependsOn = mbd.getDependsOn();
for (String dep : dependsOn) {
getBean(dep);
```

```java
if (mbd.isSingleton()) {
    sharedInstance = getSingleton(beanName, () -> {
        return createBean(beanName, mbd, args);
    });
}
```

```java
beforePrototypeCreation(beanName);
prototypeInstance = createBean(beanName, mbd, args);
bean = getObjectForBeanInstance(prototypeInstance, name, beanName, mbd);
```

```java
Object createBean (String beanName, RootBeanDefinition mbd, @Nullable Object[] args)
```

```java
RootBeanDefinition mbdToUse = mbd;
Object beanInstance = doCreateBean(beanName, mbdToUse, args);
return beanInstance;
```

```java
Object doCreateBean (String beanName, RootBeanDefinition mbd, @Nullable Object[] args)
```

```java
instanceWrapper = createBeanInstance(beanName, mbd, args);
Object bean = instanceWrapper.getWrappedInstance();
Object exposedObject = bean;
```

```java
populateBean(beanName, mbd, instanceWrapper);
exposedObject = initializeBean(beanName, exposedObject, mbd);
return exposedObject;
```

```java
BeanWrapper createBeanInstance (String beanName, RootBeanDefinition mbd, @Nullable Object[] args)
```

```java
ctors = mbd.getPreferredConstructors();
return instantiateBean(beanName, mbd);
```

```java
BeanWrapper instantiateBean (String beanName, RootBeanDefinition mbd)
```

```java
beanInstance = getInstantiationStrategy().instantiate(mbd, beanName, this);
BeanWrapper bw = new BeanWrapperImpl(beanInstance);
initBeanWrapper(bw);
return bw;
```

## 3. 创建Bean 实例
```java
Object instantiate (RootBeanDefinition bd, @Nullable String beanName, BeanFactory owner)
```

```java
Constructor<?> constructorToUse;
return BeanUtils.instantiateClass(constructorToUse);
```

```java
T instantiateClass (Constructor<T> ctor, Object... args)
```

```java
Class<?>[] parameterTypes = ctor.getParameterTypes();
Object[] argsWithDefaultValues = new Object[args.length];
return ctor.newInstance(argsWithDefaultValues);
```
