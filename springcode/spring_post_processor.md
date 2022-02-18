<!-- ---
title: PostProcessor 总结
date: 2022-02-09 07:28:43
category: java100, spring, springcode
--- -->

# PostProcessor 总结

## BeanDefinitionRegistryPostProcessor

BeanFactoryPostProcessor 的子接口，允许注册Bean Definition，在BeanFactoryPostProcessor 之前执行。

```java
public interface BeanDefinitionRegistryPostProcessor extends BeanFactoryPostProcessor {
    // 可以添加或修改应用程序上下文的内部bean工厂下的Bean Definition。所有的bean定义都将被加载，但是还没有bean被实例化,允许重写或添加属性。
    void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) throws BeansException;
}
```

## BeanFactoryPostProcessor

允许对应用程序上下文的 `bean定义` 进行自定义修改的工厂钩子，列如：自定义配置文件中配置的Bean属性。

ApplicationContext 在 `Bean定义` 中自动检测 BeanFactoryPostProcessor Bean，并在创建任何其他Bean之前应用它们。

```java
public interface BeanFactoryPostProcessor {
    // 可以修改应用程序上下文的内部bean工厂。所有的bean定义都将被加载，但是还没有bean被实例化,允许重写或添加属性。
    void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException;
}
```

## BeanPostProcessor

允许自定义修改新的bean 实例的工厂钩子函数。

ApplicationContext可以在其bean定义中自动检测BeanPostProcessor bean，并将这些后处理器应用于随后创建的任何bean。

刷新容器时注册，创建bean 对象时触发调用。

```java
public interface BeanPostProcessor {
    /**
     * 在任何bean初始化回调之前（例如InitializingBean的afterPropertiesSet或自定义init-method）之前，
     */
    @Nullable
    default Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
        return bean;
    }
    /**
     * 在任何bean初始化回调之后（例如InitializingBean的afterPropertiesSet或自定义init-method）之后，
     */
    @Nullable
    default Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
        return bean;
    }
}
```


## 参考资料

- [Spring扩展之三：BeanPostProcessor接口](https://www.cnblogs.com/lucky-yqy/p/14022474.html)
- [Spring扩展之四：BeanFactoryPostProcessor接口](https://www.cnblogs.com/lucky-yqy/p/14031852.html)
