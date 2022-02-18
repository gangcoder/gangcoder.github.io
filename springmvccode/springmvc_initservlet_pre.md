HttpServletBean
+ void init ()

FrameworkServlet
- void initServletBean ()
- WebApplicationContext initWebApplicationContext ()
- WebApplicationContext createWebApplicationContext (@Nullable ApplicationContext parent)
- void configureAndRefreshWebApplicationContext (ConfigurableWebApplicationContext wac)

DispatcherServlet
- void onRefresh (ApplicationContext context)
- void initStrategies (ApplicationContext context)
- void initLocaleResolver (ApplicationContext context)
- T getDefaultStrategy (ApplicationContext context, Class<T> strategyInterface)
- void initHandlerMappings (ApplicationContext context)
- void initHandlerAdapters (ApplicationContext context)

# DispatcherServlet 初始化
## 1. 容器初始化
```java
void init ()
```

```java
PropertyValues pvs = new ServletConfigPropertyValues(getServletConfig(), this.requiredProperties);
BeanWrapper bw = PropertyAccessorFactory.forBeanPropertyAccess(this);
bw.setPropertyValues(pvs, true);
```

```java
initServletBean();
```

```java
void initServletBean ()
```

```java
this.webApplicationContext = initWebApplicationContext();
```

```java
WebApplicationContext initWebApplicationContext ()
```

```java
wac = createWebApplicationContext(rootContext);
```

```java
WebApplicationContext createWebApplicationContext (@Nullable ApplicationContext parent)
```

```java
Class<?> contextClass = getContextClass();
ConfigurableWebApplicationContext wac = (ConfigurableWebApplicationContext) BeanUtils.instantiateClass(contextClass);
```

```java
String configLocation = getContextConfigLocation();
if (configLocation != null) {
    wac.setConfigLocation(configLocation);
}
```

```java
configureAndRefreshWebApplicationContext(wac);
```

```java
void configureAndRefreshWebApplicationContext (ConfigurableWebApplicationContext wac)
```

```java
applyInitializers(wac);
wac.refresh();
```

## 2. mvc 初始化
```java
void onRefresh (ApplicationContext context)
```

```java
initStrategies(context);
```

```java
void initStrategies (ApplicationContext context)
```

```java
initLocaleResolver(context);
```

```java
initHandlerMappings(context);
```

```java
initHandlerAdapters(context);
```

```java
void initLocaleResolver (ApplicationContext context)
```

```java
this.localeResolver = context.getBean(LOCALE_RESOLVER_BEAN_NAME, LocaleResolver.class);
```

```java
this.localeResolver = getDefaultStrategy(context, LocaleResolver.class);
```

```java
T getDefaultStrategy (ApplicationContext context, Class<T> strategyInterface)
```

```java
List<T> strategies = getDefaultStrategies(context, strategyInterface);
return strategies.get(0);
```

```java
void initHandlerMappings (ApplicationContext context)
```

```java
Map<String, HandlerMapping> matchingBeans = BeanFactoryUtils.beansOfTypeIncludingAncestors(context, HandlerMapping.class, true, false);
```

```java
void initHandlerAdapters (ApplicationContext context)
```

```java
Map<String, HandlerAdapter> matchingBeans = BeanFactoryUtils.beansOfTypeIncludingAncestors(context, HandlerAdapter.class, true, false);
```
