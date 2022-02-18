<!-- ---
title: 路由处理器与适配器解析注册
date: 2022-01-23 15:50:45
category: java100, springmvc, code
--- -->

# 路由处理器与适配器解析注册

![](draw/springmvc_requestmapping.svg)

Spring 容器创建最后，实例化所有 Bean，此时会初始化 RequestMappingHandler 和 RequestMappingHandlerAdapter。

## 1. 路由解析

RequestMappingHandler 会扫描所有 bean 带有 @RequestMapping 注解的，将其组装成 RequestMappingInfo 映射关系，然后将它注册到 mappingRegistry 变量中。之后可以通过映射关系，输入 URL 找到对应的处理器 Controller。

```java
public class RequestMappingHandlerMapping extends RequestMappingInfoHandlerMapping
		implements MatchableHandlerMapping, EmbeddedValueResolverAware {
	public void afterPropertiesSet() {
		// ...

		super.afterPropertiesSet();
	}
}

public abstract class AbstractHandlerMethodMapping<T> extends AbstractHandlerMapping implements InitializingBean {
    private final MappingRegistry mappingRegistry = new MappingRegistry();
    
    @Override
	public void afterPropertiesSet() {
		initHandlerMethods();
	}

	protected void initHandlerMethods() {
        // 获取容器中所有 bean 名字
		for (String beanName : getCandidateBeanNames()) {
			if (!beanName.startsWith(SCOPED_TARGET_NAME_PREFIX)) {
                // 遍历获取控制器方法
				processCandidateBean(beanName);
			}
		}
		// ...
	}

	protected void processCandidateBean(String beanName) {
		Class<?> beanType = null;
		try {
			beanType = obtainApplicationContext().getType(beanName);
		}
		catch (Throwable ex) {
			// ...
		}
		if (beanType != null && isHandler(beanType)) {
			detectHandlerMethods(beanName);
		}
	}

	protected void detectHandlerMethods(Object handler) {
		Class<?> handlerType = (handler instanceof String ?
				obtainApplicationContext().getType((String) handler) : handler.getClass());

		if (handlerType != null) {
			Class<?> userType = ClassUtils.getUserClass(handlerType);

            // 通过反射，获取类中所有方法
            // 筛选出 public 类型，并且带有 @RequestMapping 注解的方法
			Map<Method, T> methods = MethodIntrospector.selectMethods(userType, (MethodIntrospector.MetadataLookup<T>) method -> {
                try {
                    return getMappingForMethod(method, userType);
                }
                catch (Throwable ex) {
                    // ...
                }
            });
        
            // ...
			methods.forEach((method, mapping) -> {
				Method invocableMethod = AopUtils.selectInvocableMethod(method, userType);
                // 注册上面获取到的映射关系
				registerHandlerMethod(handler, invocableMethod, mapping);
			});
		}
	}

    // 子类调用，注册mapping
	protected void registerHandlerMethod(Object handler, Method method, T mapping) {
		this.mappingRegistry.register(mapping, handler, method);
	}
}

public class RequestMappingHandlerMapping extends RequestMappingInfoHandlerMapping
		implements MatchableHandlerMapping, EmbeddedValueResolverAware {
	@Override
	@Nullable
	protected RequestMappingInfo getMappingForMethod(Method method, Class<?> handlerType) {
        // 获取method 的 RequestMapping 注解信息
		RequestMappingInfo info = createRequestMappingInfo(method);

        // ...
        
		return info;
	}

	@Nullable
	private RequestMappingInfo createRequestMappingInfo(AnnotatedElement element) {
		RequestMapping requestMapping = AnnotatedElementUtils.findMergedAnnotation(element, RequestMapping.class);
		// ...
		return (requestMapping != null ? createRequestMappingInfo(requestMapping, condition) : null);
	}
}
```

```java
class MappingRegistry {

    private final Map<T, MappingRegistration<T>> registry = new HashMap<>();

    private final Map<T, HandlerMethod> mappingLookup = new LinkedHashMap<>();

    private final MultiValueMap<String, T> urlLookup = new LinkedMultiValueMap<>();

    public void register(T mapping, Object handler, Method method) {
        // ...
        HandlerMethod handlerMethod = createHandlerMethod(handler, method);
        this.mappingLookup.put(mapping, handlerMethod);

        List<String> directUrls = getDirectUrls(mapping);
        for (String url : directUrls) {
            this.urlLookup.add(url, mapping);
        }

        // ...
        // 将映射关系放入  Map<T, MappingRegistration<T>> registry
        this.registry.put(mapping, new MappingRegistration<>(mapping, handlerMethod, directUrls, name));

        // ...
    }
}
```

## 2. 适配器初始化

适配器初始化工具变量，用来处理 @ControllerAdvice 、InitBinder 等注解和参数。

```java
public class RequestMappingHandlerAdapter extends AbstractHandlerMethodAdapter
		implements BeanFactoryAware, InitializingBean {
	@Override
	public void afterPropertiesSet() {
		// 初始化控制器切面
		initControllerAdviceCache();

		// ...
	}
}
```

## 参考资料

- [SpringMVC源码之Controller查找原理](https://www.cnblogs.com/w-y-c-m/p/8416630.html)

