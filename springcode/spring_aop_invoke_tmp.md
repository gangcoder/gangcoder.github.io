# Spring AOP 代理实例化与调用

## Bean 初始化后置处理

```java
Object initializeBean (String beanName, Object bean, @Nullable RootBeanDefinition mbd)
```

```java
invokeAwareMethods(beanName, bean);
```

```java
Object wrappedBean = bean;
wrappedBean = applyBeanPostProcessorsBeforeInitialization(wrappedBean, beanName);
```

```java
invokeInitMethods(beanName, wrappedBean, mbd);
```

```java
wrappedBean = applyBeanPostProcessorsAfterInitialization(wrappedBean, beanName);
return wrappedBean;
```

```java
Object applyBeanPostProcessorsAfterInitialization (Object existingBean, String beanName)
```

```java
Object result = existingBean;
for (BeanPostProcessor processor : getBeanPostProcessors()) {
    Object current = processor.postProcessAfterInitialization(result, beanName);
    result = current;
}
return result;
```

## 代理对象实例化
```java
Object postProcessAfterInitialization (@Nullable Object bean, String beanName)
```

```java
return wrapIfNecessary(bean, beanName, cacheKey);
```

```java
Object wrapIfNecessary (Object bean, String beanName, Object cacheKey)
```

```java
Object[] specificInterceptors = getAdvicesAndAdvisorsForBean(bean.getClass(), beanName, null);
```

```java
this.advisedBeans.put(cacheKey, Boolean.TRUE);
Object proxy = createProxy(bean.getClass(), beanName, specificInterceptors, new SingletonTargetSource(bean));
this.proxyTypes.put(cacheKey, proxy.getClass());
return proxy;
```

```java
Object[] getAdvicesAndAdvisorsForBean (Class<?> beanClass, String beanName, @Nullable TargetSource targetSource)
```

```java
List<Advisor> advisors = findEligibleAdvisors(beanClass, beanName);
return advisors.toArray();
```

```java
List<Advisor> findEligibleAdvisors (Class<?> beanClass, String beanName)
```

```java
List<Advisor> candidateAdvisors = findCandidateAdvisors();
```

```java
List<Advisor> eligibleAdvisors = findAdvisorsThatCanApply(candidateAdvisors, beanClass, beanName);
```

```java
extendAdvisors(eligibleAdvisors);
eligibleAdvisors = sortAdvisors(eligibleAdvisors);
return eligibleAdvisors;
```

```java
List<Advisor> findCandidateAdvisors ()
```

```java
return this.advisorRetrievalHelper.findAdvisorBeans();
```

```java
List<Advisor> findAdvisorBeans ()
```

```java
advisorNames = BeanFactoryUtils.beanNamesForTypeIncludingAncestors(this.beanFactory, Advisor.class, true, false);
this.cachedAdvisorBeanNames = advisorNames;
```

```java
for (String name : advisorNames) {
    advisors.add(this.beanFactory.getBean(name, Advisor.class));
}
return advisors;
```

### 查找Bean 的切面
```java
List<Advisor> findAdvisorsThatCanApply (List<Advisor> candidateAdvisors, Class<?> beanClass, String beanName)
```

```java
return AopUtils.findAdvisorsThatCanApply(candidateAdvisors, beanClass);
```

```java
List<Advisor> findAdvisorsThatCanApply (List<Advisor> candidateAdvisors, Class<?> clazz)
```

```java
for (Advisor candidate : candidateAdvisors) {
    canApply(candidate, clazz, hasIntroductions)
}
```

```java
boolean canApply (Pointcut pc, Class<?> targetClass, boolean hasIntroductions)
```

```java
classes.addAll(ClassUtils.getAllInterfacesForClassAsSet(targetClass));
```

```java
for (Class<?> clazz : classes)
Method[] methods = ReflectionUtils.getAllDeclaredMethods(clazz);
```

```java
for (Method method : methods)
methodMatcher.matches(method, targetClass)
```

### 生成代理

```java
Object createProxy (Class<?> beanClass, @Nullable String beanName, @Nullable Object[] specificInterceptors, ...)
```

```java
ProxyFactory proxyFactory = new ProxyFactory();
proxyFactory.addAdvisors(advisors);
proxyFactory.setTargetSource(targetSource);
return proxyFactory.getProxy(getProxyClassLoader());
```

```java
Object getProxy (@Nullable ClassLoader classLoader)
```

```java
return createAopProxy().getProxy(classLoader);
```

```java
AopProxy createAopProxy ()
```

```java
return getAopProxyFactory().createAopProxy(this);
```

```java
AopProxy createAopProxy (AdvisedSupport config)
```

```java
return new ObjenesisCglibAopProxy(config);
```

```java
return new JdkDynamicAopProxy(config);
```

##  JDK 动态代理
### JDK 动态代理
```java
Object getProxy (@Nullable ClassLoader classLoader)
```

```java
Class<?>[] proxiedInterfaces = AopProxyUtils.completeProxiedInterfaces(this.advised, true);
return Proxy.newProxyInstance(classLoader, proxiedInterfaces, this);
```

### JDK 动态代理调用
```java
Object invoke (Object proxy, Method method, Object[] args)
```

```java
List<Object> chain = this.advised.getInterceptorsAndDynamicInterceptionAdvice(method, targetClass);
```

```java
MethodInvocation invocation = new ReflectiveMethodInvocation(proxy, target, method, args, targetClass, chain);
retVal = invocation.proceed();
return retVal;
```

## Cglib 代理
### Cglib 代理
```java
Object getProxy (@Nullable ClassLoader classLoader)
```

```java
Enhancer enhancer = createEnhancer();
enhancer.setSuperclass(proxySuperClass);
```

```java
Callback[] callbacks = getCallbacks(rootClass);
```

```java
enhancer.setCallbackFilter(new ProxyCallbackFilter(this.advised.getConfigurationOnlyCopy(), this.fixedInterceptorMap, this.fixedInterceptorOffset));
return createProxyClassAndInstance(enhancer, callbacks);
```

```java
Object createProxyClassAndInstance (Enhancer enhancer, Callback[] callbacks)
```

```java
enhancer.setCallbacks(callbacks);
return (this.constructorArgs != null && this.constructorArgTypes != null ? enhancer.create(this.constructorArgTypes, this.constructorArgs) : enhancer.create());
```

```java
Callback[] getCallbacks (Class<?> rootClass)
```

```java
targetInterceptor = new DynamicUnadvisedInterceptor(this.advised.getTargetSource()));
```

```java
Callback[] mainCallbacks = new Callback[] { aopInterceptor, targetInterceptor, ... };
callbacks = mainCallbacks;
return callbacks;
```

### Cglib 代理调用
```java
Object intercept (Object proxy, Method method, Object[] args, ...)
```

```java
List<Object> chain = this.advised.getInterceptorsAndDynamicInterceptionAdvice(method, targetClass);
retVal = new CglibMethodInvocation(proxy, target, method, args, targetClass, chain, methodProxy).proceed();
return retVal;
```

```java
Object proceed ()
```

```java
return super.proceed();
```

### 链式递归调用
```java
List<Object> getInterceptorsAndDynamicInterceptionAdvice (Method method, @Nullable Class<?> targetClass)
```

```java
cached = this.advisorChainFactory.getInterceptorsAndDynamicInterceptionAdvice(this, method, targetClass);
```

```java
Object proceed ()
```

```java
if (this.currentInterceptorIndex == this.interceptorsAndDynamicMethodMatchers.size() - 1) {
    return invokeJoinpoint();
}
```

```java
Object interceptorOrInterceptionAdvice = this.interceptorsAndDynamicMethodMatchers.get(++this.currentInterceptorIndex);
return ((MethodInterceptor) interceptorOrInterceptionAdvice).invoke(this);
```

```java
Object invoke (MethodInvocation mi)
```

```java
try {
    return mi.proceed();
} finally {
    invokeAdviceMethod(getJoinPointMatch(), null, null);
}
```

```java
Object invoke (MethodInvocation mi)
```

```java
this.advice.before(mi.getMethod(), mi.getArguments(), mi.getThis());
return mi.proceed();
```
