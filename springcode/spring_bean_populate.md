<!-- ---
title: Bean 属性注入
date: 2021-12-30 09:49:48
category: java100, spring, springcode
--- -->

# Bean 属性注入

![](draw/spring_bean_populate.drawio.svg)

## 1. 属性注入


```java
// populateBean(beanName, mbd, instanceWrapper);
// org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory
public abstract class AbstractAutowireCapableBeanFactory extends AbstractBeanFactory
		implements AutowireCapableBeanFactory {
	protected void populateBean(String beanName, RootBeanDefinition mbd, @Nullable BeanWrapper bw) {
		// ...

		PropertyValues pvs = (mbd.hasPropertyValues() ? mbd.getPropertyValues() : null);

		// ...
		if (pvs != null) {
			applyPropertyValues(beanName, mbd, bw, pvs);
		}
	}

	protected void applyPropertyValues(String beanName, BeanDefinition mbd, BeanWrapper bw, PropertyValues pvs) {
		// ...
		
		// 属性值深拷贝
		List<PropertyValue> deepCopy = new ArrayList<>(original.size());
		for (PropertyValue pv : original) {
			if (pv.isConverted()) {
				// ...
			}
			else {
				String propertyName = pv.getName();
				Object originalValue = pv.getValue();
				
				// 属性值转换
				Object resolvedValue = valueResolver.resolveValueIfNecessary(pv, originalValue);
				Object convertedValue = resolvedValue;
				
				// ...
				if (resolvedValue == originalValue) {
					// ...
				}
				else {
					deepCopy.add(new PropertyValue(pv, convertedValue));
				}
			}
		}
		
		// ...
		try {
			bw.setPropertyValues(new MutablePropertyValues(deepCopy));
		}
		catch (BeansException ex) {
			// ...
		}
	}
}
```


```java
// org.springframework.beans.AbstractPropertyAccessor
public abstract class AbstractPropertyAccessor extends TypeConverterSupport implements ConfigurablePropertyAccessor {
	@Override
	public void setPropertyValues(PropertyValues pvs) throws BeansException {
		setPropertyValues(pvs, false, false);
	}

	@Override
	public void setPropertyValues(PropertyValues pvs, boolean ignoreUnknown, boolean ignoreInvalid)
			throws BeansException {

		// ...
		try {
			for (PropertyValue pv : propertyValues) {
				try {
					setPropertyValue(pv);
				}
				catch (PropertyAccessException ex) {
					// ...
				}
			}
		}
		finally {
			// ...
		}
	}
}
```

```java
// org.springframework.beans.AbstractNestablePropertyAccessor
public abstract class AbstractNestablePropertyAccessor extends AbstractPropertyAccessor {
	@Override
	public void setPropertyValue(PropertyValue pv) throws BeansException {
		PropertyTokenHolder tokens = (PropertyTokenHolder) pv.resolvedTokens;
		if (tokens == null) {
			String propertyName = pv.getName();

			try {
				nestedPa = getPropertyAccessorForPropertyPath(propertyName);
			}
			catch (NotReadablePropertyException ex) {
				// ...
			}
			tokens = getPropertyNameTokens(getFinalPath(nestedPa, propertyName));
			
			// ...
			nestedPa.setPropertyValue(tokens, pv);
		}
	}

	protected void setPropertyValue(PropertyTokenHolder tokens, PropertyValue pv) throws BeansException {
		if (tokens.keys != null) {
			// ...
		}
		else {
			processLocalProperty(tokens, pv);
		}
	}

	private void processLocalProperty(PropertyTokenHolder tokens, PropertyValue pv) {
		PropertyHandler ph = getLocalPropertyHandler(tokens.actualName);
		// ...

		Object oldValue = null;
		try {
			Object originalValue = pv.getValue();
			Object valueToApply = originalValue;
			
			// ...
			ph.setValue(valueToApply);
		}
		catch (TypeMismatchException ex) {
			// ...
		}
	}
}
```

```java
// org.springframework.beans.BeanWrapperImpl.BeanPropertyHandler
class BeanPropertyHandler extends PropertyHandler {
    @Override
    public void setValue(@Nullable Object value) throws Exception {
        Method writeMethod = (this.pd instanceof GenericTypeAwarePropertyDescriptor ?
                ((GenericTypeAwarePropertyDescriptor) this.pd).getWriteMethodForActualAccess() :
                this.pd.getWriteMethod());
        if (System.getSecurityManager() != null) {
            // ...
        }
        else {
            ReflectionUtils.makeAccessible(writeMethod);
            writeMethod.invoke(getWrappedInstance(), value);
        }
    }
}
```

## 2. Aware注入

使用Spring的时候我们将自己的Bean实现BeanNameAware接口、BeanFactoryAware接口等，依赖容器帮我们注入到当前Bean。

```java
// exposedObject = initializeBean(beanName, exposedObject, mbd);

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

		// ...
		try {
			invokeInitMethods(beanName, wrappedBean, mbd);
		}
		catch (Throwable ex) {
			// ...
		}

		return wrappedBean;
	}

	private void invokeAwareMethods(String beanName, Object bean) {
		if (bean instanceof Aware) {
			if (bean instanceof BeanNameAware) {
				((BeanNameAware) bean).setBeanName(beanName);
			}
			
			// ...
		}
	}
}
```

### 调用初始化方法

1. 先判断Bean是否InitializingBean的实现类，是的话，将Bean强转为InitializingBean，直接调用afterPropertiesSet()方法
2. 尝试去拿init-method，假如有的话，通过反射，调用initMethod

```java
// org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory
public abstract class AbstractAutowireCapableBeanFactory extends AbstractBeanFactory
		implements AutowireCapableBeanFactory {

	protected void invokeInitMethods(String beanName, Object bean, @Nullable RootBeanDefinition mbd)
			throws Throwable {

		boolean isInitializingBean = (bean instanceof InitializingBean);
		if (isInitializingBean && (mbd == null || !mbd.isExternallyManagedInitMethod("afterPropertiesSet"))) {
			if (System.getSecurityManager() != null) {
				// ...
			}
			else {
				((InitializingBean) bean).afterPropertiesSet();
			}
		}

		if (mbd != null && bean.getClass() != NullBean.class) {
			String initMethodName = mbd.getInitMethodName();
			if (StringUtils.hasLength(initMethodName) &&
					!(isInitializingBean && "afterPropertiesSet".equals(initMethodName)) &&
					!mbd.isExternallyManagedInitMethod(initMethodName)) {
				invokeCustomInitMethod(beanName, bean, mbd);
			}
		}
	}

	protected void invokeCustomInitMethod(String beanName, Object bean, RootBeanDefinition mbd)
			throws Throwable {

		String initMethodName = mbd.getInitMethodName();

		Method initMethod = (mbd.isNonPublicAccessAllowed() ?
				BeanUtils.findMethod(bean.getClass(), initMethodName) :
				ClassUtils.getMethodIfAvailable(bean.getClass(), initMethodName));

		// ...
		Method methodToInvoke = ClassUtils.getInterfaceMethodIfPossible(initMethod);

		if (System.getSecurityManager() != null) {
			// ...
		}
		else {
			try {
				methodToInvoke.invoke(bean);
			}
			catch (InvocationTargetException ex) {
				// ...
			}
		}
	}
}    
```

## 3. 注册需要执行销毁方法的Bean


```java
// registerDisposableBeanIfNecessary(beanName, bean, mbd);

// org.springframework.beans.factory.support.AbstractBeanFactory
public abstract class AbstractBeanFactory extends FactoryBeanRegistrySupport implements ConfigurableBeanFactory {
	protected void registerDisposableBeanIfNecessary(String beanName, Object bean, RootBeanDefinition mbd) {
		AccessControlContext acc = (System.getSecurityManager() != null ? getAccessControlContext() : null);
		if (!mbd.isPrototype() && requiresDestruction(bean, mbd)) {
			if (mbd.isSingleton()) {
				registerDisposableBean(beanName,
						new DisposableBeanAdapter(bean, beanName, mbd, getBeanPostProcessors(), acc));
			}
			else {
				// ...
			}
		}
	}
}
```

```java
// org.springframework.beans.factory.support.DefaultSingletonBeanRegistry
public class DefaultSingletonBeanRegistry extends SimpleAliasRegistry implements SingletonBeanRegistry {
    private final Map<String, Object> disposableBeans = new LinkedHashMap<>();

	public void registerDisposableBean(String beanName, DisposableBean bean) {
		synchronized (this.disposableBeans) {
			this.disposableBeans.put(beanName, bean);
		}
	}
}
```

## 参考资料

- [【Spring源码分析】非懒加载的单例Bean初始化过程（下篇）](https://www.cnblogs.com/xrq730/p/6363055.html)

