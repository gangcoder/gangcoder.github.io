<!-- ---
title: XML 应用上下文启动实现
date: 2021-12-28 12:27:19
category: java100, spring, springcode
--- -->

# XML 应用上下文启动实现

![](draw/spring_xml_application_context.svg)

## 1. 创建应用

ClassPathXmlApplicationContext 中触发刷新整个Spring上下文信息。

```java
// 创建 Spring 应用
// ApplicationContext ac = new ClassPathXmlApplicationContext("classpath:applicationContext.xml");

// org.springframework.context.support.ClassPathXmlApplicationContext
public class ClassPathXmlApplicationContext extends AbstractXmlApplicationContext {
    public ClassPathXmlApplicationContext(String configLocation) throws BeansException {
        this(new String[] {configLocation}, true, null);
    }

    public ClassPathXmlApplicationContext(
            String[] configLocations, boolean refresh, @Nullable ApplicationContext parent)
            throws BeansException {

        super(parent);

        // 设置配置文件地址
        setConfigLocations(configLocations);
        if (refresh) {
            // 刷新整个Spring上下文信息
            refresh();
        }
    }
}
```

## 2. 应用刷新

Refresh 方法实现Spring 应用启动处理。

```java
// org.springframework.context.support.AbstractApplicationContext
public abstract class AbstractApplicationContext extends DefaultResourceLoader
        implements ConfigurableApplicationContext {
    @Override
    public void refresh() throws BeansException, IllegalStateException {
        synchronized (this.startupShutdownMonitor) {
            prepareRefresh();

            ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

            prepareBeanFactory(beanFactory);

            try {
                postProcessBeanFactory(beanFactory);

                invokeBeanFactoryPostProcessors(beanFactory);

                registerBeanPostProcessors(beanFactory);

                initMessageSource();

                initApplicationEventMulticaster();

                onRefresh();

                registerListeners();

                finishBeanFactoryInitialization(beanFactory);

                finishRefresh();
            }
            catch (BeansException ex) {
                // ...
            }
        }
    }
}
```

## 3. 刷新前置处理

准备刷新Spring上下文。

```java
// org.springframework.context.support.AbstractApplicationContext
public abstract class AbstractApplicationContext extends DefaultResourceLoader
        implements ConfigurableApplicationContext {
    protected void prepareRefresh() {
        this.startupDate = System.currentTimeMillis();
        this.closed.set(false);
        this.active.set(true);

        // ...

        initPropertySources();

        // ...
    }
}
```

## 4. 创建 BeanFactory

obtainFreshBeanFactory 方法获取 Bean工厂。


```java
// ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

// org.springframework.context.support.AbstractApplicationContext
public abstract class AbstractApplicationContext extends DefaultResourceLoader
        implements ConfigurableApplicationContext {
    protected ConfigurableListableBeanFactory obtainFreshBeanFactory() {
        refreshBeanFactory();
        return getBeanFactory();
    }
}

// org.springframework.context.support.AbstractRefreshableApplicationContext
public abstract class AbstractRefreshableApplicationContext extends AbstractApplicationContext {
    @Override
    protected final void refreshBeanFactory() throws BeansException {
        // ...
        try {
            // 创建bean 工厂
            DefaultListableBeanFactory beanFactory = createBeanFactory();
            beanFactory.setSerializationId(getId());
            customizeBeanFactory(beanFactory);

            // 加载bean 定义
            loadBeanDefinitions(beanFactory);
            this.beanFactory = beanFactory;
        }
        catch (IOException ex) {
            // ...
        }
    }

    protected DefaultListableBeanFactory createBeanFactory() {
        return new DefaultListableBeanFactory(getInternalParentBeanFactory());
    }

    @Override
    public final ConfigurableListableBeanFactory getBeanFactory() {
        DefaultListableBeanFactory beanFactory = this.beanFactory;
        // ...
        return beanFactory;
    }
}
```

加载 BeanDefinition 定义。

```java
// org.springframework.context.support.AbstractXmlApplicationContext
public abstract class AbstractXmlApplicationContext extends AbstractRefreshableConfigApplicationContext {
    @Override
    protected void loadBeanDefinitions(DefaultListableBeanFactory beanFactory) throws BeansException, IOException {
        // bean 定义读取器
        XmlBeanDefinitionReader beanDefinitionReader = new XmlBeanDefinitionReader(beanFactory);

        // 属性设置
        beanDefinitionReader.setEnvironment(this.getEnvironment());
        beanDefinitionReader.setResourceLoader(this);
        beanDefinitionReader.setEntityResolver(new ResourceEntityResolver(this));
        initBeanDefinitionReader(beanDefinitionReader);        

        // 加载bean 定义
        loadBeanDefinitions(beanDefinitionReader);
    }
}
```

## 参考资料

- [【Spring源码分析】Bean加载流程概览](https://www.cnblogs.com/xrq730/p/6285358.html)

