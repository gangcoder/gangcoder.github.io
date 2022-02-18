AbstractAutowireCapableBeanFactory
- void populateBean (String beanName, RootBeanDefinition mbd, @Nullable BeanWrapper bw)
- void applyPropertyValues (String beanName, BeanDefinition mbd, BeanWrapper bw, ...)
- Object initializeBean (String beanName, Object bean, @Nullable RootBeanDefinition mbd)
- void invokeAwareMethods (String beanName, Object bean)
- void invokeInitMethods (String beanName, Object bean, @Nullable RootBeanDefinition mbd)
- void invokeCustomInitMethod (String beanName, Object bean, RootBeanDefinition mbd)

AbstractPropertyAccessor
+ void setPropertyValues (PropertyValues pvs)
+ void setPropertyValues (PropertyValues pvs, boolean ignoreUnknown, boolean ignoreInvalid)

AbstractNestablePropertyAccessor
+ void setPropertyValue (PropertyValue pv)
- void setPropertyValue (PropertyTokenHolder tokens, PropertyValue pv)
- void processLocalProperty (PropertyTokenHolder tokens, PropertyValue pv)

BeanPropertyHandler
+ void setValue (@Nullable Object value)

AbstractBeanFactory
- void registerDisposableBeanIfNecessary (String beanName, Object bean, RootBeanDefinition mbd)

DefaultSingletonBeanRegistry
- final Map<String, Object> disposableBeans = new LinkedHashMap<>()
--
+ void registerDisposableBean (String beanName, DisposableBean bean)

# Bean 属性注入
## 1. 属性注入
```java
void populateBean (String beanName, RootBeanDefinition mbd, @Nullable BeanWrapper bw)
```

```java
PropertyValues pvs = (mbd.hasPropertyValues() ? mbd.getPropertyValues() : null);
applyPropertyValues(beanName, mbd, bw, pvs);
```

```java
void applyPropertyValues (String beanName, BeanDefinition mbd, BeanWrapper bw, ...)
```

```java
List<PropertyValue> deepCopy = new ArrayList<>(original.size());
for (PropertyValue pv : original) {
    Object originalValue = pv.getValue();
    Object resolvedValue = valueResolver.resolveValueIfNecessary(pv, originalValue);
    Object convertedValue = resolvedValue;
    deepCopy.add(new PropertyValue(pv, convertedValue));
}
```

```java
bw.setPropertyValues(new MutablePropertyValues(deepCopy));
```

```java
void setPropertyValues (PropertyValues pvs)
```

```java
setPropertyValues(pvs, false, false);
```

```java
void setPropertyValues (PropertyValues pvs, boolean ignoreUnknown, boolean ignoreInvalid)
```

```java
for (PropertyValue pv : propertyValues) {
    setPropertyValue(pv);
}
```

```java
void setPropertyValue (PropertyValue pv)
```

```java
tokens = getPropertyNameTokens(getFinalPath(nestedPa, propertyName));
nestedPa.setPropertyValue(tokens, pv);
```

```java
void setPropertyValue (PropertyTokenHolder tokens, PropertyValue pv)
```

```java
processLocalProperty(tokens, pv);
```

```java
void processLocalProperty (PropertyTokenHolder tokens, PropertyValue pv)
```

```java
PropertyHandler ph = getLocalPropertyHandler(tokens.actualName);
Object originalValue = pv.getValue();
Object valueToApply = originalValue;
ph.setValue(valueToApply);
```

```java
void setValue (@Nullable Object value)
```

```java
Method writeMethod = this.pd.getWriteMethod();
ReflectionUtils.makeAccessible(writeMethod);
writeMethod.invoke(getWrappedInstance(), value);
```

## 2. Aware注入
```java
Object initializeBean (String beanName, Object bean, @Nullable RootBeanDefinition mbd)
```

```java
invokeAwareMethods(beanName, bean);
```

```java
invokeInitMethods(beanName, wrappedBean, mbd);
return wrappedBean;
```

```java
void invokeAwareMethods (String beanName, Object bean)
```

```java
if (bean instanceof Aware) {
    if (bean instanceof BeanNameAware) {
        ((BeanNameAware) bean).setBeanName(beanName);
    }
}
```

### 调用初始化方法
```java
void invokeInitMethods (String beanName, Object bean, @Nullable RootBeanDefinition mbd)
```

```java
boolean isInitializingBean = (bean instanceof InitializingBean);
if (isInitializingBean ) {
    ((InitializingBean) bean).afterPropertiesSet();
}
```

```java
if (mbd != null && bean.getClass() != NullBean.class) {
    String initMethodName = mbd.getInitMethodName();
    invokeCustomInitMethod(beanName, bean, mbd);
}
```

```java
void invokeCustomInitMethod (String beanName, Object bean, RootBeanDefinition mbd)
```

```java
Method methodToInvoke = ClassUtils.getInterfaceMethodIfPossible(initMethod);
methodToInvoke.invoke(bean);
```

## 3. 注册需要执行销毁方法的Bean
```java
void registerDisposableBeanIfNecessary (String beanName, Object bean, RootBeanDefinition mbd)
```

```java
registerDisposableBean(beanName, new DisposableBeanAdapter(bean, beanName, mbd, getBeanPostProcessors(), acc));
```

```java
void registerDisposableBean (String beanName, DisposableBean bean)
```

```java
synchronized (this.disposableBeans) {
    this.disposableBeans.put(beanName, bean);
}
```
