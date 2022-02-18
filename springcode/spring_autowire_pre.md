AbstractAutowireCapableBeanFactory
- void populateBean (String beanName, RootBeanDefinition mbd, @Nullable BeanWrapper bw)
- void autowireByName (String beanName, AbstractBeanDefinition mbd, BeanWrapper bw, ...)
- void autowireByType (String beanName, AbstractBeanDefinition mbd, BeanWrapper bw, ...)

DefaultListableBeanFactory
+ Object resolveDependency (DependencyDescriptor descriptor, @Nullable String requestingBeanName, @Nullable Set<String> autowiredBeanNames, ...)
+ Object doResolveDependency (DependencyDescriptor descriptor, @Nullable String beanName, @Nullable Set<String> autowiredBeanNames, ...)
- Map<String, Object> findAutowireCandidates (@Nullable String beanName, Class<?> requiredType, DependencyDescriptor descriptor)
- void addCandidateEntry (Map<String, Object> candidates, String candidateName, DependencyDescriptor descriptor, ...)
+ Object resolveCandidate (String beanName, Class<?> requiredType, BeanFactory beanFactory)



public abstract class AbstractAutowireCapableBeanFactory {
    protected void populateBean(String beanName, RootBeanDefinition mbd, @Nullable BeanWrapper bw);
    protected void autowireByName(String beanName, AbstractBeanDefinition mbd, BeanWrapper bw, MutablePropertyValues pvs);
    protected void autowireByType(String beanName, AbstractBeanDefinition mbd, BeanWrapper bw, MutablePropertyValues pvs);
}

public class DefaultListableBeanFactory {
    public Object resolveDependency(DependencyDescriptor descriptor, @Nullable String requestingBeanName, @Nullable Set<String> autowiredBeanNames, @Nullable TypeConverter typeConverter) throws BeansException;
    public Object doResolveDependency(DependencyDescriptor descriptor, @Nullable String beanName, @Nullable Set<String> autowiredBeanNames, @Nullable TypeConverter typeConverter) throws BeansException;
    protected Map<String, Object> findAutowireCandidates(@Nullable String beanName, Class<?> requiredType, DependencyDescriptor descriptor);
    private void addCandidateEntry(Map<String, Object> candidates, String candidateName, DependencyDescriptor descriptor, Class<?> requiredType);
    public Object resolveCandidate(String beanName, Class<?> requiredType, BeanFactory beanFactory) throws BeansException;
}

# 自动装配实现
## 1. byName 源码实现
```java
void populateBean (String beanName, RootBeanDefinition mbd, @Nullable BeanWrapper bw)
```

```java
if (resolvedAutowireMode == AUTOWIRE_BY_NAME) {
    autowireByName(beanName, mbd, bw, newPvs);
}
```

```java
if (resolvedAutowireMode == AUTOWIRE_BY_TYPE) {
    autowireByType(beanName, mbd, bw, newPvs);
}
```

```java
void autowireByName (String beanName, AbstractBeanDefinition mbd, BeanWrapper bw, ...)
```

```java
String[] propertyNames = unsatisfiedNonSimpleProperties(mbd, bw);
for (String propertyName : propertyNames)
```

```java
if (containsBean(propertyName)) {
    Object bean = getBean(propertyName);
    pvs.add(propertyName, bean);
    registerDependentBean(propertyName, beanName);
}
```

## 2. byType 源码实现
```java
void autowireByType (String beanName, AbstractBeanDefinition mbd, BeanWrapper bw, ...)
```

```java
for (String propertyName : propertyNames) {
    Object autowiredArgument = resolveDependency(desc, beanName, autowiredBeanNames, converter);
    if (autowiredArgument != null) {
        pvs.add(propertyName, autowiredArgument);
    }
}
```

```java
Object resolveDependency (DependencyDescriptor descriptor, @Nullable String requestingBeanName, @Nullable Set<String> autowiredBeanNames, ...)
```

```java
result = doResolveDependency(descriptor, requestingBeanName, autowiredBeanNames, typeConverter);
```

```java
Object doResolveDependency (DependencyDescriptor descriptor, @Nullable String beanName, @Nullable Set<String> autowiredBeanNames, ...)
```

```java
Map<String, Object> matchingBeans = findAutowireCandidates(beanName, type, descriptor);
Map.Entry<String, Object> entry = matchingBeans.entrySet().iterator().next();
instanceCandidate = entry.getValue();
Object result = instanceCandidate;
return result;
```

```java
Map<String, Object> findAutowireCandidates (@Nullable String beanName, Class<?> requiredType, DependencyDescriptor descriptor)
```

```java
String[] candidateNames = BeanFactoryUtils.beanNamesForTypeIncludingAncestors(this, requiredType, true, descriptor.isEager());
for (String candidate : candidateNames) {
    if (!isSelfReference(beanName, candidate) && isAutowireCandidate(candidate, descriptor)) {
        addCandidateEntry(result, candidate, descriptor, requiredType);
    }
}
```

```java
void addCandidateEntry (Map<String, Object> candidates, String candidateName, DependencyDescriptor descriptor, ...)
```

```java
Object beanInstance = descriptor.resolveCandidate(candidateName, requiredType, this);
candidates.put(candidateName, (beanInstance instanceof NullBean ? null : beanInstance));
```

```java
Object resolveCandidate (String beanName, Class<?> requiredType, BeanFactory beanFactory)
```

```java
return beanFactory.getBean(beanName);
```
