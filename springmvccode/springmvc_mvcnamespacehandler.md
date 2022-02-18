<!-- ---
title: Spring MVC 标签解析
date: 2022-01-29 07:56:42
category: java100, springmvc, code
--- -->

# Spring MVC 标签解析

## 1. 注解驱动解析

此时会代码中注册相关 bean 的 BeanDefinition。

```java
class AnnotationDrivenBeanDefinitionParser implements BeanDefinitionParser {
	@Override
	@Nullable
	public BeanDefinition parse(Element element, ParserContext context) {
		// ...
        // 注册 RequestMappingHandlerMapping BeanDefinition
		RootBeanDefinition handlerMappingDef = new RootBeanDefinition(RequestMappingHandlerMapping.class);
		handlerMappingDef.setSource(source);
		handlerMappingDef.setRole(BeanDefinition.ROLE_INFRASTRUCTURE);
		handlerMappingDef.getPropertyValues().add("order", 0);
		handlerMappingDef.getPropertyValues().add("contentNegotiationManager", contentNegotiationManager);
        readerContext.getRegistry().registerBeanDefinition(HANDLER_MAPPING_BEAN_NAME, handlerMappingDef);

        // ...
        // 注册 ConfigurableWebBindingInitializer BeanDefinition
		RootBeanDefinition handlerAdapterDef = new RootBeanDefinition(RequestMappingHandlerAdapter.class);
		handlerAdapterDef.setSource(source);
		handlerAdapterDef.setRole(BeanDefinition.ROLE_INFRASTRUCTURE);
		handlerAdapterDef.getPropertyValues().add("contentNegotiationManager", contentNegotiationManager);
		handlerAdapterDef.getPropertyValues().add("webBindingInitializer", bindingDef);
		handlerAdapterDef.getPropertyValues().add("messageConverters", messageConverters);
		readerContext.getRegistry().registerBeanDefinition(HANDLER_ADAPTER_BEAN_NAME, handlerAdapterDef);

        // ...
		return null;
	}
}
```

## 参考资料

- []()