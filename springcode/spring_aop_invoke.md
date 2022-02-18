<!-- ---
title: Spring AOP 代理实例化与调用
date: 2021-12-19 22:05:31
category: java100, springcode
--- -->

# Spring AOP 代理实例化与调用

`AspectJAwareAdvisorAutoProxyCreator` 是Spring AOP的核心类，这个类实现了 `类/接口-->代理` 的转换过程。

AOP 代理实例化与调用逻辑：

1. AspectJAwareAdvisorAutoProxyCreator 是BeanPostProcessor 接口的实现类
2. postProcessAfterInitialization 方法实现在父类 AbstractAutoProxyCreator 中
3. bean 初始化后，调用 postProcessAfterInitialization 方法生成代理

Bean 生成代理的时机：在每个Bean初始化之后，如果需要，调用 AspectJAwareAdvisorAutoProxyCreator 中的 postProcessAfterInitialization 为Bean生成代理。

## Bean 初始化后置处理

Bean 初始化后，逐个调用 BeanPostProcessor 后置处理逻辑 `postProcessAfterInitialization`。

```java
// org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory
public abstract class AbstractAutowireCapableBeanFactory extends AbstractBeanFactory
		implements AutowireCapableBeanFactory {
	protected Object initializeBean(String beanName, Object bean, @Nullable RootBeanDefinition mbd) {
		if (System.getSecurityManager() != null) {
			// ...
		}
		else {
			invokeAwareMethods(beanName, bean);
		}

		Object wrappedBean = bean;
		if (mbd == null || !mbd.isSynthetic()) {
			wrappedBean = applyBeanPostProcessorsBeforeInitialization(wrappedBean, beanName);
		}

		try {
			// 初始化处理
			invokeInitMethods(beanName, wrappedBean, mbd);
		}
		catch (Throwable ex) {
			// ...
		}
		if (mbd == null || !mbd.isSynthetic()) {
			// bean 初始化后，调用 BeanPostProcessor 后置处理
			wrappedBean = applyBeanPostProcessorsAfterInitialization(wrappedBean, beanName);
		}

		return wrappedBean;
	}

    @Override
	public Object applyBeanPostProcessorsAfterInitialization(Object existingBean, String beanName) throws BeansException {

		Object result = existingBean;
		for (BeanPostProcessor processor : getBeanPostProcessors()) {
			// 最终会调用 aop 自动代理类的 bean 初始化后置处理
			Object current = processor.postProcessAfterInitialization(result, beanName);
			if (current == null) {
				return result;
			}
			result = current;
		}
		return result;
	}
}
```

## 代理对象实例化

bean 初始化后触发自动代理类处理逻辑。

```java
// org.springframework.aop.aspectj.autoproxy.AspectJAwareAdvisorAutoProxyCreator
// org.springframework.aop.framework.autoproxy.AbstractAdvisorAutoProxyCreator
// org.springframework.aop.framework.autoproxy.AbstractAutoProxyCreator
public abstract class AbstractAutoProxyCreator extends ProxyProcessorSupport
		implements SmartInstantiationAwareBeanPostProcessor, BeanFactoryAware {
	@Override
	public Object postProcessAfterInitialization(@Nullable Object bean, String beanName) {
		if (bean != null) {
			Object cacheKey = getCacheKey(bean.getClass(), beanName);
			if (this.earlyProxyReferences.remove(cacheKey) != bean) {
				return wrapIfNecessary(bean, beanName, cacheKey);
			}
		}
		return bean;
	}
}

// org.springframework.aop.framework.autoproxy.AbstractAutoProxyCreator
public abstract class AbstractAutoProxyCreator extends ProxyProcessorSupport
		implements SmartInstantiationAwareBeanPostProcessor, BeanFactoryAware {

	protected Object wrapIfNecessary(Object bean, String beanName, Object cacheKey) {
		// 不需要生成代理的场景判断，如内部类，已生成代理等情况
		// ...

		// 获取bean 对应的 Advice；不是每个bean 都需要生成代理
		Object[] specificInterceptors = getAdvicesAndAdvisorsForBean(bean.getClass(), beanName, null);
		if (specificInterceptors != DO_NOT_PROXY) {
			this.advisedBeans.put(cacheKey, Boolean.TRUE);

			// 创建bean 对象的代理对象，并返回；这一步，完成了bean 到代理类的生成
			Object proxy = createProxy(
					bean.getClass(), beanName, specificInterceptors, new SingletonTargetSource(bean));
			this.proxyTypes.put(cacheKey, proxy.getClass());
			return proxy;
		}

		this.advisedBeans.put(cacheKey, Boolean.FALSE);
		return bean;
	}
}

// org.springframework.aop.framework.autoproxy.AbstractAdvisorAutoProxyCreator
public abstract class AbstractAdvisorAutoProxyCreator extends AbstractAutoProxyCreator {

    @Override
	@Nullable
	protected Object[] getAdvicesAndAdvisorsForBean(
			Class<?> beanClass, String beanName, @Nullable TargetSource targetSource) {

		// 找到bean 对象对应的 advisor 配置
		List<Advisor> advisors = findEligibleAdvisors(beanClass, beanName);
		if (advisors.isEmpty()) {
			return DO_NOT_PROXY;
		}
		return advisors.toArray();
	}

	protected List<Advisor> findEligibleAdvisors(Class<?> beanClass, String beanName) {
		// 获取所有 advisor
		List<Advisor> candidateAdvisors = findCandidateAdvisors();

		// 找到当前bean 对应的 advisor
		List<Advisor> eligibleAdvisors = findAdvisorsThatCanApply(candidateAdvisors, beanClass, beanName);

		// 添加默认 advisor 
		extendAdvisors(eligibleAdvisors);
		if (!eligibleAdvisors.isEmpty()) {
			// 有多个 advisor 时，排序
			eligibleAdvisors = sortAdvisors(eligibleAdvisors);
		}
		return eligibleAdvisors;
	}

	// 获取所有 advisor
	protected List<Advisor> findCandidateAdvisors() {
		// ...
		return this.advisorRetrievalHelper.findAdvisorBeans();
	}
}
```

从 IOC 容器中获取所有 Advisor.class 类型的 bean 。

```java
// org.springframework.aop.framework.autoproxy.BeanFactoryAdvisorRetrievalHelper
public class BeanFactoryAdvisorRetrievalHelper {
	public List<Advisor> findAdvisorBeans() {
		// 找到 IOC 容器中 Advisor.class 类型的所有 bean 名称
		String[] advisorNames = this.cachedAdvisorBeanNames;
		if (advisorNames == null) {
			// ...
			advisorNames = BeanFactoryUtils.beanNamesForTypeIncludingAncestors(
					this.beanFactory, Advisor.class, true, false);
			this.cachedAdvisorBeanNames = advisorNames;
		}

		// 如果没有 Advisor.class 类型 的bean，直接返回
		if (advisorNames.length == 0) {
			return new ArrayList<>();
		}

		List<Advisor> advisors = new ArrayList<>();
		for (String name : advisorNames) {
			if (isEligibleBean(name)) {
				if (this.beanFactory.isCurrentlyInCreation(name)) {
					// ...
				}
				else {
					try {
						// 从IOC 容器中，获取 bean 对象
						advisors.add(this.beanFactory.getBean(name, Advisor.class));
					}
					catch (BeanCreationException ex) {
						// ...
						throw ex;
					}
				}
			}
		}
		return advisors;
	}
}
```

### 查找Bean 的切面

取到所有 advisor 后，找到特定 bean 对应的 advisor。

```java
// org.springframework.aop.framework.autoproxy.AbstractAdvisorAutoProxyCreator
public abstract class AbstractAdvisorAutoProxyCreator extends AbstractAutoProxyCreator {

	protected List<Advisor> findAdvisorsThatCanApply(
			List<Advisor> candidateAdvisors, Class<?> beanClass, String beanName) {
		// ...
		try {
			return AopUtils.findAdvisorsThatCanApply(candidateAdvisors, beanClass);
		}
		finally {
			// ...
		}
	}
}
```

```java
// org.springframework.aop.support.AopUtils
public abstract class AopUtils {
	// 具体查找逻辑实现
    public static List<Advisor> findAdvisorsThatCanApply(List<Advisor> candidateAdvisors, Class<?> clazz) {
		// ...

		List<Advisor> eligibleAdvisors = new ArrayList<>();
		// ...

		boolean hasIntroductions = !eligibleAdvisors.isEmpty();
		for (Advisor candidate : candidateAdvisors) {
			if (candidate instanceof IntroductionAdvisor) {
				// already processed
				continue;
			}

			// 逻辑围绕 canApply 展开
			if (canApply(candidate, clazz, hasIntroductions)) {
				eligibleAdvisors.add(candidate);
			}
		}
		return eligibleAdvisors;
	}

    public static boolean canApply(Advisor advisor, Class<?> targetClass, boolean hasIntroductions) {
		if (advisor instanceof IntroductionAdvisor) {
			return ((IntroductionAdvisor) advisor).getClassFilter().matches(targetClass);
		}
		else if (advisor instanceof PointcutAdvisor) {
			// 取到切入点表达式结构
			PointcutAdvisor pca = (PointcutAdvisor) advisor;
			return canApply(pca.getPointcut(), targetClass, hasIntroductions);
		}
		else {
			return true;
		}
	}

	// 目标类必须满足expression的匹配规则
	// 目标类中的方法必须满足expression的匹配规则，当然这里方法不是全部需要满足expression的匹配规则，有一个方法满足即可
	public static boolean canApply(Pointcut pc, Class<?> targetClass, boolean hasIntroductions) {
		// 类过滤器匹配
		if (!pc.getClassFilter().matches(targetClass)) {
			return false;
		}

		// ...

		// 方法匹配器
		IntroductionAwareMethodMatcher introductionAwareMethodMatcher = null;
		if (methodMatcher instanceof IntroductionAwareMethodMatcher) {
			introductionAwareMethodMatcher = (IntroductionAwareMethodMatcher) methodMatcher;
		}

		Set<Class<?>> classes = new LinkedHashSet<>();
		// ...
		// 为了取到类的所有方法，需要先找到类继承的所有父类或者接口，用于提取所有方法
		classes.addAll(ClassUtils.getAllInterfacesForClassAsSet(targetClass));

		for (Class<?> clazz : classes) {
			// 提取所有方法
			Method[] methods = ReflectionUtils.getAllDeclaredMethods(clazz);
			for (Method method : methods) {
				// 逐个方法匹配，看是否有需要代理的方法
				if (introductionAwareMethodMatcher != null ?
						introductionAwareMethodMatcher.matches(method, targetClass, hasIntroductions) :
						methodMatcher.matches(method, targetClass)) {
					return true;
				}
			}
		}

		return false;
	}
}
```

### 生成代理

```java
// org.springframework.aop.framework.autoproxy.AbstractAutoProxyCreator
public abstract class AbstractAutoProxyCreator extends ProxyProcessorSupport
		implements SmartInstantiationAwareBeanPostProcessor, BeanFactoryAware {

	protected Object createProxy(Class<?> beanClass, @Nullable String beanName,
			@Nullable Object[] specificInterceptors, TargetSource targetSource) {

		// ...

		// 创建代理工厂，由代理工厂生成代理对象
		ProxyFactory proxyFactory = new ProxyFactory();
		proxyFactory.copyFrom(this);

		// ...

		// 设置代理工厂属性
		Advisor[] advisors = buildAdvisors(beanName, specificInterceptors);
		proxyFactory.addAdvisors(advisors);
		proxyFactory.setTargetSource(targetSource);

		// ...

		// 生成代理对象
		return proxyFactory.getProxy(getProxyClassLoader());
	}

}

// org.springframework.aop.framework.ProxyFactory
public class ProxyFactory extends ProxyCreatorSupport {
	public Object getProxy(@Nullable ClassLoader classLoader) {
		// 创建AopProxy接口实现类
		// 获取代理
		return createAopProxy().getProxy(classLoader);
	}
}

// org.springframework.aop.framework.ProxyCreatorSupport
public class ProxyCreatorSupport extends AdvisedSupport {
	protected final synchronized AopProxy createAopProxy() {
		// ...
		return getAopProxyFactory().createAopProxy(this);
	}
}

// org.springframework.aop.framework.DefaultAopProxyFactory
public class DefaultAopProxyFactory implements AopProxyFactory, Serializable {
	@Override
	public AopProxy createAopProxy(AdvisedSupport config) throws AopConfigException {
		// 创建代理对象实例
		if (config.isOptimize() || config.isProxyTargetClass() || hasNoUserSuppliedProxyInterfaces(config)) {
			Class<?> targetClass = config.getTargetClass();
			// ...
			if (targetClass.isInterface() || Proxy.isProxyClass(targetClass)) {
				return new JdkDynamicAopProxy(config);
			}

			// 使用CGLIB
			return new ObjenesisCglibAopProxy(config);
		}
		else {
			// 使用 JDK 动态代理
			return new JdkDynamicAopProxy(config);
		}
	}
}
```

##  JDK 动态代理

### JDK 动态代理

```java
// org.springframework.aop.framework.JdkDynamicAopProxy
final class JdkDynamicAopProxy implements AopProxy, InvocationHandler, Serializable {
    public JdkDynamicAopProxy(AdvisedSupport config) throws AopConfigException {
		// ...
		this.advised = config;
	}

	// 获取代理
    @Override
	public Object getProxy(@Nullable ClassLoader classLoader) {
		// 获取所有接口
		Class<?>[] proxiedInterfaces = AopProxyUtils.completeProxiedInterfaces(this.advised, true);
		
		// 创建原生 jdk 动态代理
		return Proxy.newProxyInstance(classLoader, proxiedInterfaces, this);
	}
}
```

### JDK 动态代理调用

JdkDynamicAopProxy本身实现了InvocationHandler接口，因此具体代理前后处理的逻辑在invoke方法中:

```java
// org.springframework.aop.framework.JdkDynamicAopProxy
final class JdkDynamicAopProxy implements AopProxy, InvocationHandler, Serializable {
	@Override
	@Nullable
	public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
		Object oldProxy = null;
		boolean setProxyContext = false;

		TargetSource targetSource = this.advised.targetSource;
		Object target = null;

		try {
			// ...

			Object retVal;

			// ...
			target = targetSource.getTarget();
			Class<?> targetClass = (target != null ? target.getClass() : null);

			// 获取调用链
			List<Object> chain = this.advised.getInterceptorsAndDynamicInterceptionAdvice(method, targetClass);

			if (chain.isEmpty()) {
				// ...
			}
			else {
				// 链式调用触发
				MethodInvocation invocation =
						new ReflectiveMethodInvocation(proxy, target, method, args, targetClass, chain);
				retVal = invocation.proceed();
			}

			// ...
			return retVal;
		}
		finally {
			// ...
		}
	}
}
```

## Cglib 代理

### Cglib 代理

```java
// org.springframework.aop.framework.ObjenesisCglibAopProxy
class ObjenesisCglibAopProxy extends CglibAopProxy {
	public ObjenesisCglibAopProxy(AdvisedSupport config) {
		super(config);
	}
}

// org.springframework.aop.framework.CglibAopProxy
class CglibAopProxy implements AopProxy, Serializable {
    @Override
	public Object getProxy(@Nullable ClassLoader classLoader) {
		try {
			Class<?> rootClass = this.advised.getTargetClass();

			// ...

			Class<?> proxySuperClass = rootClass;

			// ...

			// 创建 CGLIB Enhancer
			Enhancer enhancer = createEnhancer();
			// ...
			enhancer.setSuperclass(proxySuperClass);
			enhancer.setInterfaces(AopProxyUtils.completeProxiedInterfaces(this.advised));
			enhancer.setNamingPolicy(SpringNamingPolicy.INSTANCE);
			enhancer.setStrategy(new ClassLoaderAwareGeneratorStrategy(classLoader));

			Callback[] callbacks = getCallbacks(rootClass);
			
			// ...

			// enhancer 设置
			enhancer.setCallbackFilter(new ProxyCallbackFilter(
					this.advised.getConfigurationOnlyCopy(), this.fixedInterceptorMap, this.fixedInterceptorOffset));
			
			// 生成代理对象
			return createProxyClassAndInstance(enhancer, callbacks);
		}
		catch (CodeGenerationException | IllegalArgumentException ex) {
			// ...
		}
	}

    protected Object createProxyClassAndInstance(Enhancer enhancer, Callback[] callbacks) {
		enhancer.setInterceptDuringConstruction(false);
		enhancer.setCallbacks(callbacks);
		return (this.constructorArgs != null && this.constructorArgTypes != null ?
				enhancer.create(this.constructorArgTypes, this.constructorArgs) :
				enhancer.create());
	}

	// 配置拦截器
	private Callback[] getCallbacks(Class<?> rootClass) throws Exception {
		// aop 功能的拦截器
		Callback aopInterceptor = new DynamicAdvisedInterceptor(this.advised);

		// 目标类业务拦截器
		Callback targetInterceptor;
		if (exposeProxy) {
			// ...
		}
		else {
			targetInterceptor = (isStatic ?
					new StaticUnadvisedInterceptor(this.advised.getTargetSource().getTarget()) :
					new DynamicUnadvisedInterceptor(this.advised.getTargetSource()));
		}

		// ...

		// 拦截器组

		Callback[] mainCallbacks = new Callback[] {
				aopInterceptor,  // for normal advice
				targetInterceptor,  // invoke target without considering advice, if optimized
				new SerializableNoOp(),  // no override for methods mapped to this
				targetDispatcher, this.advisedDispatcher,
				new EqualsInterceptor(this.advised),
				new HashCodeInterceptor(this.advised)
		};

		Callback[] callbacks;

		if (isStatic && isFrozen) {
			// ...
		}
		else {
			callbacks = mainCallbacks;
		}
		return callbacks;
	}

}
```

### Cglib 代理调用

cglib 在调用时，通过 MethodInterceptor 触发。

```java
// org.springframework.aop.framework.CglibAopProxy.DynamicAdvisedInterceptor
class DynamicAdvisedInterceptor implements MethodInterceptor, Serializable {

    @Override
    @Nullable
    public Object intercept(Object proxy, Method method, Object[] args, MethodProxy methodProxy) throws Throwable {
        // ...
        try {
            // ...
			
			// 获取调用链
            target = targetSource.getTarget();
            Class<?> targetClass = (target != null ? target.getClass() : null);
            List<Object> chain = this.advised.getInterceptorsAndDynamicInterceptionAdvice(method, targetClass);
            Object retVal;
            
			// ...
            if (chain.isEmpty() && Modifier.isPublic(method.getModifiers())) {
                // ...
            }
            else {
                // 触发链式调用
                retVal = new CglibMethodInvocation(proxy, target, method, args, targetClass, chain, methodProxy).proceed();
            }
            
			// ...
            return retVal;
        }
        finally {
            // ...
        }
    }
}

// 触发 ReflectiveMethodInvocation 的链式调用
// org.springframework.aop.framework.CglibAopProxy.CglibMethodInvocation
class CglibMethodInvocation extends ReflectiveMethodInvocation {
    @Override
    @Nullable
    public Object proceed() throws Throwable {
        try {
            return super.proceed();
        }
        catch (RuntimeException ex) {
            throw ex;
        }
        catch (Exception ex) {
            // ...
        }
    }
}
```

### 链式递归调用

```java
// List<Object> chain = this.advised.getInterceptorsAndDynamicInterceptionAdvice(method, targetClass);

// org.springframework.aop.framework.AdvisedSupport
public class AdvisedSupport extends ProxyConfig implements Advised {
	// 获取调用链
    public List<Object> getInterceptorsAndDynamicInterceptionAdvice(Method method, @Nullable Class<?> targetClass) {
		MethodCacheKey cacheKey = new MethodCacheKey(method);
		List<Object> cached = this.methodCache.get(cacheKey);
		if (cached == null) {
			cached = this.advisorChainFactory.getInterceptorsAndDynamicInterceptionAdvice(
					this, method, targetClass);
			this.methodCache.put(cacheKey, cached);
		}
		return cached;
	}

}
```

```java
// MethodInvocation invocation = new ReflectiveMethodInvocation(proxy, target, method, args, targetClass, chain);
// retVal = invocation.proceed();

// org.springframework.aop.framework.ReflectiveMethodInvocation
public class ReflectiveMethodInvocation implements ProxyMethodInvocation, Cloneable {
    protected ReflectiveMethodInvocation(
			Object proxy, @Nullable Object target, Method method, @Nullable Object[] arguments,
			@Nullable Class<?> targetClass, List<Object> interceptorsAndDynamicMethodMatchers) {

		this.proxy = proxy;
		this.target = target;
		this.targetClass = targetClass;
		this.method = BridgeMethodResolver.findBridgedMethod(method);
		this.arguments = AopProxyUtils.adaptArgumentsIfNecessary(method, arguments);
		this.interceptorsAndDynamicMethodMatchers = interceptorsAndDynamicMethodMatchers;
	}

	@Override
	@Nullable
	public Object proceed() throws Throwable {
		// 到最后一个时退出处理
		if (this.currentInterceptorIndex == this.interceptorsAndDynamicMethodMatchers.size() - 1) {
			return invokeJoinpoint();
		}
		
		// 逐个取出处理器，进行处理
		Object interceptorOrInterceptionAdvice =
				this.interceptorsAndDynamicMethodMatchers.get(++this.currentInterceptorIndex);
		if (interceptorOrInterceptionAdvice instanceof InterceptorAndDynamicMethodMatcher) {
			// ...
		}
		else {
			// 调用处理器处理
			return ((MethodInterceptor) interceptorOrInterceptionAdvice).invoke(this);
		}
	}
}

// org.springframework.aop.aspectj.AspectJAfterAdvice
public class AspectJAfterAdvice extends AbstractAspectJAdvice {
    @Override
	public Object invoke(MethodInvocation mi) throws Throwable {
		try {
			// 迭代调用触发
			return mi.proceed();
		}
		finally {
			// after 处理器逻辑调用
			invokeAdviceMethod(getJoinPointMatch(), null, null);
		}
	}
}

// org.springframework.aop.framework.adapter.MethodBeforeAdviceInterceptor
public class MethodBeforeAdviceInterceptor implements MethodInterceptor, BeforeAdvice, Serializable {

	@Override
	public Object invoke(MethodInvocation mi) throws Throwable {
		// before 处理逻辑调用
		this.advice.before(mi.getMethod(), mi.getArguments(), mi.getThis());

		// 迭代调用触发
		return mi.proceed();
	}

}
```

## 参考资料

- [【Spring源码分析】AOP源码解析（下篇）](https://www.cnblogs.com/xrq730/p/6757608.html)
