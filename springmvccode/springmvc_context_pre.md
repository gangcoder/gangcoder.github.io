ContextLoaderListener
+ void contextInitialized (ServletContextEvent event)

ContextLoader
- static final Properties defaultStrategies
--
+ WebApplicationContext initWebApplicationContext (ServletContext servletContext)
- WebApplicationContext createWebApplicationContext (ServletContext sc)
- Class<?> determineContextClass (ServletContext servletContext)
- void configureAndRefreshWebApplicationContext (ConfigurableWebApplicationContext wac, ServletContext sc)


# 初始化 WebApplicationContext
## 1. ContextLoaderListener
```java
void contextInitialized (ServletContextEvent event)
```

```java
initWebApplicationContext(event.getServletContext());
```

```java
WebApplicationContext initWebApplicationContext (ServletContext servletContext)
```

```java
this.context = createWebApplicationContext(servletContext);
```

```java
configureAndRefreshWebApplicationContext(cwac, servletContext);
```

```java
servletContext.setAttribute(WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE, this.context);
return this.context;
```

```java
WebApplicationContext createWebApplicationContext (ServletContext sc)
```

```java
Class<?> contextClass = determineContextClass(sc);
return (ConfigurableWebApplicationContext) BeanUtils.instantiateClass(contextClass);
```

```java
Class<?> determineContextClass (ServletContext servletContext)
```

```java
contextClassName = defaultStrategies.getProperty(WebApplicationContext.class.getName());
return ClassUtils.forName(contextClassName, ContextLoader.class.getClassLoader());
```

## 2. refresh
```java
void configureAndRefreshWebApplicationContext (ConfigurableWebApplicationContext wac, ServletContext sc)
```

```java
wac.setServletContext(sc);
String configLocationParam = sc.getInitParameter(CONFIG_LOCATION_PARAM);
if (configLocationParam != null) {
    wac.setConfigLocation(configLocationParam);
}
customizeContext(sc, wac);
```

```java
wac.refresh();
```
