RequestMappingHandlerMapping
+ void afterPropertiesSet ()
- RequestMappingInfo getMappingForMethod (Method method, Class<?> handlerType)
- RequestMappingInfo createRequestMappingInfo (AnnotatedElement element)

AbstractHandlerMethodMapping
- final MappingRegistry mappingRegistry = new MappingRegistry()
--
+ void afterPropertiesSet ()
- void initHandlerMethods ()
- void processCandidateBean (String beanName)
- void detectHandlerMethods (Object handler)
- void registerHandlerMethod (Object handler, Method method, T mapping)

MappingRegistry
- final Map<T, MappingRegistration<T>> registry = new HashMap<>()
- final Map<T, HandlerMethod> mappingLookup = new LinkedHashMap<>()
- final MultiValueMap<String, T> urlLookup = new LinkedMultiValueMap<>()
--
+ void register (T mapping, Object handler, Method method)

RequestMappingHandlerAdapter
+ void afterPropertiesSet ()



# 路由处理器与适配器解析注册
## 1. 路由解析
```java
void afterPropertiesSet ()
```

```java
super.afterPropertiesSet();
```

```java
void afterPropertiesSet ()
```

```java
initHandlerMethods();
```

```java
void initHandlerMethods ()
```

```java
for (String beanName : getCandidateBeanNames()) {
    processCandidateBean(beanName);
}
```

```java
void processCandidateBean (String beanName)
```

```java
beanType = obtainApplicationContext().getType(beanName);
detectHandlerMethods(beanName);
```

```java
void detectHandlerMethods (Object handler)
```

```java
Map<Method, T> methods = MethodIntrospector.selectMethods(userType, (MethodIntrospector.MetadataLookup<T>) method -> {
    return getMappingForMethod(method, userType);
});
```

```java
methods.forEach((method, mapping) -> {
    Method invocableMethod = AopUtils.selectInvocableMethod(method, userType);
    registerHandlerMethod(handler, invocableMethod, mapping);
});
```

```java
void registerHandlerMethod (Object handler, Method method, T mapping)
```

```java
this.mappingRegistry.register(mapping, handler, method);
```

```java
RequestMappingInfo getMappingForMethod (Method method, Class<?> handlerType)
```

```java
RequestMappingInfo info = createRequestMappingInfo(method);
return info;
```

```java
RequestMappingInfo createRequestMappingInfo (AnnotatedElement element)
```

```java
RequestMapping requestMapping = AnnotatedElementUtils.findMergedAnnotation(element, RequestMapping.class);
return (requestMapping != null ? createRequestMappingInfo(requestMapping, condition) : null);
```

```java
void register (T mapping, Object handler, Method method)
```

```java
HandlerMethod handlerMethod = createHandlerMethod(handler, method);
this.mappingLookup.put(mapping, handlerMethod);
```

```java
List<String> directUrls = getDirectUrls(mapping);
for (String url : directUrls) {
    this.urlLookup.add(url, mapping);
}
```

```java
this.registry.put(mapping, new MappingRegistration<>(mapping, handlerMethod, directUrls, name));
```
