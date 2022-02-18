<!-- ---
title: Bean 实例化过程
date: 2021-09-07 09:09:52
category: java100, springcode
--- -->

# Bean 实例化过程

## 示例代码

```java
public class MultiFunctionBean implements InitializingBean, BeanNameAware, BeanClassLoaderAware {
    private int    propertyA;

    private int    propertyB;

    public int getPropertyA() {
        return propertyA;
    }

    public void setPropertyA(int propertyA) {
        this.propertyA = propertyA;
    }

    public int getPropertyB() {
        return propertyB;
    }

    public void setPropertyB(int propertyB) {
        this.propertyB = propertyB;
    }

    public void initMethod() {
        System.out.println("Enter MultiFunctionBean.initMethod()");
    }

    @Override
    public void setBeanClassLoader(ClassLoader classLoader) {
        System.out.println("Enter MultiFunctionBean.setBeanClassLoader(ClassLoader classLoader)");
    }

    @Override
    public void setBeanName(String name) {
        System.out.println("Enter MultiFunctionBean.setBeanName(String name)");
    }

    @Override
    public void afterPropertiesSet() throws Exception {
        System.out.println("Enter MultiFunctionBean.afterPropertiesSet()");
    }

    @Override
    public String toString() {
        return "MultiFunctionBean [propertyA=" + propertyA + ", propertyB=" + propertyB + "]";
    }

}
```

## 1. preInstantiateSingletons 方法

refresh 方法中，finishBeanFactoryInitialization 方法用于对所有非懒加载的Bean 初始化。

```java
// finishBeanFactoryInitialization
// org.springframework.context.support.AbstractApplicationContext
public abstract class AbstractApplicationContext extends DefaultResourceLoader
		implements ConfigurableApplicationContext {

	protected void finishBeanFactoryInitialization(ConfigurableListableBeanFactory beanFactory) {
		// ...

		// 初始化所有非懒加载的Bean
		beanFactory.preInstantiateSingletons();
	}

}
```

```java
// org.springframework.beans.factory.support.DefaultListableBeanFactory
public class DefaultListableBeanFactory extends AbstractAutowireCapableBeanFactory
		implements ConfigurableListableBeanFactory, BeanDefinitionRegistry, Serializable {
	@Override
	public void preInstantiateSingletons() throws BeansException {
		// ...
		List<String> beanNames = new ArrayList<>(this.beanDefinitionNames);

		// 遍历所有bean name，进行初始化
		for (String beanName : beanNames) {
			RootBeanDefinition bd = getMergedLocalBeanDefinition(beanName);
			if (!bd.isAbstract() && bd.isSingleton() && !bd.isLazyInit()) {
				if (isFactoryBean(beanName)) {
					// ...
				}
				else {
					getBean(beanName);
				}
			}
		}

		// ...
	}
}
```

## 2. doGetBean 方法构造 Bean

```java
// org.springframework.beans.factory.support.AbstractBeanFactory
public abstract class AbstractBeanFactory extends FactoryBeanRegistrySupport implements ConfigurableBeanFactory {
	@Override
	public Object getBean(String name) throws BeansException {
		return doGetBean(name, null, null, false);
	}

	protected <T> T doGetBean(
			String name, @Nullable Class<T> requiredType, @Nullable Object[] args, boolean typeCheckOnly)
			throws BeansException {
        String beanName = transformedBeanName(name);
		Object bean;

		if (sharedInstance != null && args == null) {
			// ...
		}
		else {
			// ...

			try {
				RootBeanDefinition mbd = getMergedLocalBeanDefinition(beanName);
				checkMergedBeanDefinition(mbd, beanName, args);

				// depends-on 依赖的Bean 会优先加载
				String[] dependsOn = mbd.getDependsOn();
				if (dependsOn != null) {
					for (String dep : dependsOn) {
						// 检查是否循环依赖
						if (isDependent(beanName, dep)) {
							// ...
						}

						registerDependentBean(dep, beanName);
						try {
							getBean(dep);
						}
						catch (NoSuchBeanDefinitionException ex) {
							// ...
						}
					}
				}

				if (mbd.isSingleton()) {
					sharedInstance = getSingleton(beanName, () -> {
						try {
							return createBean(beanName, mbd, args);
						}
						catch (BeansException ex) {
							// ...
						}
					});
					bean = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
				}

				else if (mbd.isPrototype()) {
					Object prototypeInstance = null;
					try {
						beforePrototypeCreation(beanName);
						prototypeInstance = createBean(beanName, mbd, args);
					}
					finally {
						// ...
					}
					bean = getObjectForBeanInstance(prototypeInstance, name, beanName, mbd);
				}

				else {
					// ...
				}
			}
			catch (BeansException ex) {
				// ...
			}
		}

		// ...
		return (T) bean;
    }
}
```

```java
// org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory
public abstract class AbstractAutowireCapableBeanFactory extends AbstractBeanFactory
		implements AutowireCapableBeanFactory {

	@Override
	protected Object createBean(String beanName, RootBeanDefinition mbd, @Nullable Object[] args)
			throws BeanCreationException {
		RootBeanDefinition mbdToUse = mbd;

		// ...

		try {
			Object beanInstance = doCreateBean(beanName, mbdToUse, args);
			// ...
			return beanInstance;
		}
		catch (BeanCreationException | ImplicitlyAppearedSingletonException ex) {
			// ...
		}
	}

    protected Object doCreateBean(String beanName, RootBeanDefinition mbd, @Nullable Object[] args)
			throws BeanCreationException {

		BeanWrapper instanceWrapper = null;
		// ...

		if (instanceWrapper == null) {
			instanceWrapper = createBeanInstance(beanName, mbd, args);
		}

		Object bean = instanceWrapper.getWrappedInstance();
		
		// ...
		Object exposedObject = bean;
		try {

			// 填充属性
			populateBean(beanName, mbd, instanceWrapper);
			exposedObject = initializeBean(beanName, exposedObject, mbd);
		}
		catch (Throwable ex) {
			// ...
		}

		// ...
		return exposedObject;
	}

	protected BeanWrapper createBeanInstance(String beanName, RootBeanDefinition mbd, @Nullable Object[] args) {
		// ...
		ctors = mbd.getPreferredConstructors();
		
		return instantiateBean(beanName, mbd);
	}

    protected BeanWrapper instantiateBean(String beanName, RootBeanDefinition mbd) {
		try {
			Object beanInstance;
			if (System.getSecurityManager() != null) {
				// ...
			}
			else {
				beanInstance = getInstantiationStrategy().instantiate(mbd, beanName, this);
			}

			BeanWrapper bw = new BeanWrapperImpl(beanInstance);
			initBeanWrapper(bw);
			return bw;
		}
		catch (Throwable ex) {
			// ...
		}
	}
}
```

## 3. 创建Bean 实例

```java
// org.springframework.beans.factory.support.SimpleInstantiationStrategy
public class SimpleInstantiationStrategy implements InstantiationStrategy {
	@Override
	public Object instantiate(RootBeanDefinition bd, @Nullable String beanName, BeanFactory owner) {
		if (!bd.hasMethodOverrides()) {
			Constructor<?> constructorToUse;
			synchronized (bd.constructorArgumentLock) {
				constructorToUse = (Constructor<?>) bd.resolvedConstructorOrFactoryMethod;
				
				// ...
			}
			return BeanUtils.instantiateClass(constructorToUse);
		}
		else {
			// ...
		}
	}
}
```

```java
// org.springframework.beans.BeanUtils

public abstract class BeanUtils {
	public static <T> T instantiateClass(Constructor<T> ctor, Object... args) throws BeanInstantiationException {
		try {
			if (KotlinDetector.isKotlinReflectPresent() && KotlinDetector.isKotlinType(ctor.getDeclaringClass())) {
				// ...
			}
			else {
				Class<?>[] parameterTypes = ctor.getParameterTypes();
				Object[] argsWithDefaultValues = new Object[args.length];
				
				// ...
				return ctor.newInstance(argsWithDefaultValues);
			}
		}
		catch (InstantiationException ex) {
			// ...
		}
	}
}
```


## 参考资料

- [【Spring源码分析】非懒加载的单例Bean初始化过程（上篇）](https://www.cnblogs.com/xrq730/p/6361578.html)