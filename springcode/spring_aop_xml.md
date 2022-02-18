<!-- ---
title: Spring AOP 基于XML 启用原理
date: 2021-12-19 22:05:31
category: java100, springcode
--- -->

# Spring AOP 基于XML 启用原理

## 示例代码准备

```java
// 接口
public interface Dao {
	public void select();
	public void insert();
}

// 实现
public class DaoImpl implements Dao {
	@Override
	public void select() {
		System.out.println("Enter DaoImpl.select()");
	}

	@Override
	public void insert() {
		System.out.println("Enter DaoImpl.insert()");
	}
}

// 横切逻辑
public class TimeHandler {
	public void printTime() {
		System.out.println("CurrentTime:" + System.currentTimeMillis());
	}
}

// main 测试代码
Dao dao = (Dao) ac.getBean("daoImpl");
dao.select();
```

xml 配置AOP: 

```xml
<aop:config proxy-target-class="true">
    <aop:aspect id="time" ref="timeHandler">
        <aop:pointcut id="addAllMethod" expression="execution(* com.hello.Dao.*(..))"/>
        <aop:before method="printTime" pointcut-ref="addAllMethod"/>
        <aop:after method="printTime" pointcut-ref="addAllMethod"/>
    </aop:aspect>
</aop:config>
```

## 1. aop 标签解析器

基于xml 启用aop 步骤：

1. 定义xml
2. 注册aop 标签解析 Handler
3. 解析xml 遇到aop 标签时使用 aop 标签 Handler
4. 找到具体的aop 标签解析器并执行解析。


### 1.1 aop 标签解析触发

解析xml 文件遇到 `aop` 标签时使用 aop 标签解析 Handler。

```java
// org.springframework.beans.factory.xml.DefaultBeanDefinitionDocumentReader
public class DefaultBeanDefinitionDocumentReader implements BeanDefinitionDocumentReader {
	protected void parseBeanDefinitions(Element root, BeanDefinitionParserDelegate delegate) {
		if (delegate.isDefaultNamespace(root)) {
			NodeList nl = root.getChildNodes();
			for (int i = 0; i < nl.getLength(); i++) {
				Node node = nl.item(i);
				if (node instanceof Element) {
					Element ele = (Element) node;
					if (delegate.isDefaultNamespace(ele)) {
						parseDefaultElement(ele, delegate);
					}
					else {
						// 非默认标签，则进行自定义标签解析
						delegate.parseCustomElement(ele);
					}
				}
			}
		}
		// ...
	}
}

// org.springframework.beans.factory.xml.BeanDefinitionParserDelegate
public class BeanDefinitionParserDelegate {
	@Nullable
	public BeanDefinition parseCustomElement(Element ele) {
		return parseCustomElement(ele, null);
	}

	@Nullable
	public BeanDefinition parseCustomElement(Element ele, @Nullable BeanDefinition containingBd) {
		String namespaceUri = getNamespaceURI(ele);
		
        // ...
		// 根据 namespace 找到aop 标签的解析处理器
		NamespaceHandler handler = this.readerContext.getNamespaceHandlerResolver().resolve(namespaceUri);
		
        // ...
		// 执行aop 标签解析
		return handler.parse(ele, new ParserContext(this.readerContext, this, containingBd));
	}
}
```

获取 aop 标签解析 NamespaceHandler ：

```java
// org.springframework.beans.factory.xml.DefaultNamespaceHandlerResolver
public class DefaultNamespaceHandlerResolver implements NamespaceHandlerResolver {

	@Override
	public NamespaceHandler resolve(String namespaceUri) {
		// 先查找 NamespaceHandler 实例
		Map<String, Object> handlerMappings = getHandlerMappings();
		Object handlerOrClassName = handlerMappings.get(namespaceUri);
		if (handlerOrClassName == null) {
			return null;
		}
		else if (handlerOrClassName instanceof NamespaceHandler) {
			// 如果 NamespaceHandler 实例已经存在，则直接返回
			return (NamespaceHandler) handlerOrClassName;
		}
		else {
			String className = (String) handlerOrClassName;
			try {
				// 否则创建 NamespaceHandler 实例
				Class<?> handlerClass = ClassUtils.forName(className, this.classLoader);
				
				// ...
				NamespaceHandler namespaceHandler = (NamespaceHandler) BeanUtils.instantiateClass(handlerClass);

				// NamespaceHandler 初始化调用
				namespaceHandler.init();

				// 将 NamespaceHandler 存入内存缓存，后续不再重复初始化
				handlerMappings.put(namespaceUri, namespaceHandler);
				return namespaceHandler;
			}
			catch (ClassNotFoundException ex) {
				// ...
			}
		}
	}
}
```

### 1.2 aop 解析器注册与调用

初始化 NamespaceHandler 后，首先注册aop 子标签解析 Handler。

```java
// org.springframework.aop.config.AopNamespaceHandler
public class AopNamespaceHandler extends NamespaceHandlerSupport {
	@Override
	public void init() {
        // 重点关注config
		registerBeanDefinitionParser("config", new ConfigBeanDefinitionParser());
		registerBeanDefinitionParser("aspectj-autoproxy", new AspectJAutoProxyBeanDefinitionParser());
		// ...
	}
}

// org.springframework.beans.factory.xml.NamespaceHandlerSupport
public abstract class NamespaceHandlerSupport implements NamespaceHandler {
	// 存储标签解析器
	private final Map<String, BeanDefinitionParser> parsers = new HashMap<>();

	// namespace handler 注册具体标签解析器
	protected final void registerBeanDefinitionParser(String elementName, BeanDefinitionParser parser) {
		this.parsers.put(elementName, parser);
	}

	// 找到具体标签对应的解析器，并且调用解析
	@Override
	@Nullable
	public BeanDefinition parse(Element element, ParserContext parserContext) {
		BeanDefinitionParser parser = findParserForElement(element, parserContext);
		return (parser != null ? parser.parse(element, parserContext) : null);
	}

	// 从map 获取标签解析器
	@Nullable
	private BeanDefinitionParser findParserForElement(Element element, ParserContext parserContext) {
		String localName = parserContext.getDelegate().getLocalName(element);
		BeanDefinitionParser parser = this.parsers.get(localName);
		
		// ...
		return parser;
	}
}
```

## 2. 解析处理

解析 config 及其子标签，生成 RootBeanDefinition 并且注册到 IOC 容器中。

```xml
<aop:config proxy-target-class="true">
    <aop:aspect id="time" ref="timeHandler">
        <aop:pointcut id="addAllMethod" expression="execution(* com.hello.Dao.*(..))"/>
        <aop:before method="printTime" pointcut-ref="addAllMethod"/>
        <aop:after method="printTime" pointcut-ref="addAllMethod"/>
    </aop:aspect>
</aop:config>
```

config 使用 ConfigBeanDefinitionParser 进行解析:

```java
// org.springframework.aop.config.ConfigBeanDefinitionParser
class ConfigBeanDefinitionParser implements BeanDefinitionParser {
	@Override
	@Nullable
	public BeanDefinition parse(Element element, ParserContext parserContext) {
		// ...

		// AOP 代理生成器Bean 注册
		configureAutoProxyCreator(parserContext, element);

		List<Element> childElts = DomUtils.getChildElements(element);
		for (Element elt: childElts) {
			String localName = parserContext.getDelegate().getLocalName(elt);
			if (POINTCUT.equals(localName)) {
				parsePointcut(elt, parserContext);
			}
			// ...
			else if (ASPECT.equals(localName)) {
				// 逐个解析aspect 切面
				parseAspect(elt, parserContext);
			}
		}

		// ...
		return null;
	}
}
```

### 2.1 注册 AspectJAwareAdvisorAutoProxyCreator

```java
// org.springframework.aop.config.ConfigBeanDefinitionParser
class ConfigBeanDefinitionParser implements BeanDefinitionParser {
	private void configureAutoProxyCreator(ParserContext parserContext, Element element) {
		AopNamespaceUtils.registerAspectJAutoProxyCreatorIfNecessary(parserContext, element);
	}
}

// org.springframework.aop.config.AopNamespaceUtils
public abstract class AopNamespaceUtils {
	public static void registerAspectJAutoProxyCreatorIfNecessary(
			ParserContext parserContext, Element sourceElement) {

		BeanDefinition beanDefinition = AopConfigUtils.registerAspectJAutoProxyCreatorIfNecessary(
				parserContext.getRegistry(), parserContext.extractSource(sourceElement));
		useClassProxyingIfNecessary(parserContext.getRegistry(), sourceElement);
		registerComponentIfNecessary(beanDefinition, parserContext);
	}
}

// org.springframework.aop.config.AopConfigUtils
public abstract class AopConfigUtils {
	@Nullable
	public static BeanDefinition registerAspectJAutoProxyCreatorIfNecessary(
			BeanDefinitionRegistry registry, @Nullable Object source) {

		return registerOrEscalateApcAsRequired(AspectJAwareAdvisorAutoProxyCreator.class, registry, source);
	}

    	@Nullable
	private static BeanDefinition registerOrEscalateApcAsRequired(
			Class<?> cls, BeanDefinitionRegistry registry, @Nullable Object source) {

		// ...

		// 将 AspectJAwareAdvisorAutoProxyCreator 自动代理实例生成器注册到 IOC 容器
        // AUTO_PROXY_CREATOR_BEAN_NAME = "org.springframework.aop.config.internalAutoProxyCreator";
		RootBeanDefinition beanDefinition = new RootBeanDefinition(cls);
		beanDefinition.setSource(source);
		beanDefinition.getPropertyValues().add("order", Ordered.HIGHEST_PRECEDENCE);
		beanDefinition.setRole(BeanDefinition.ROLE_INFRASTRUCTURE);
		registry.registerBeanDefinition(AUTO_PROXY_CREATOR_BEAN_NAME, beanDefinition);
		return beanDefinition;
	}
}
```

### 2.2 aspect 标签解析

```xml
<aop:aspect id="time" ref="timeHandler">
	<aop:pointcut id="addAllMethod" expression="execution(* com.hello.Dao.*(..))"/>
	<aop:before method="printTime" pointcut-ref="addAllMethod"/>
	<aop:after method="printTime" pointcut-ref="addAllMethod"/>
</aop:aspect>
```

xml 中 aspect 标签解析。

```java
// org.springframework.aop.config.ConfigBeanDefinitionParser
class ConfigBeanDefinitionParser implements BeanDefinitionParser {
    	private void parseAspect(Element aspectElement, ParserContext parserContext) {
		String aspectId = aspectElement.getAttribute(ID);
		String aspectName = aspectElement.getAttribute(REF);

		try {
			List<BeanDefinition> beanDefinitions = new ArrayList<>();
			List<BeanReference> beanReferences = new ArrayList<>();

			// ...
			
			NodeList nodeList = aspectElement.getChildNodes();
			
			// 逐个解析aspect 纸标签
			for (int i = 0; i < nodeList.getLength(); i++) {
				Node node = nodeList.item(i);
				if (isAdviceNode(node, parserContext)) {
					
					// ...

					// 解析增强器
					AbstractBeanDefinition advisorDefinition = parseAdvice(
							aspectName, i, aspectElement, (Element) node, parserContext, beanDefinitions, beanReferences);
					beanDefinitions.add(advisorDefinition);
				}
			}

			AspectComponentDefinition aspectComponentDefinition = createAspectComponentDefinition(
					aspectElement, aspectId, beanDefinitions, beanReferences, parserContext);
			parserContext.pushContainingComponent(aspectComponentDefinition);

			List<Element> pointcuts = DomUtils.getChildElementsByTagName(aspectElement, POINTCUT);
			for (Element pointcutElement : pointcuts) {
				parsePointcut(pointcutElement, parserContext);
			}

			parserContext.popAndRegisterContainingComponent();
		}
		finally {
			this.parseState.pop();
		}
	}

	// 检查是否是 before, after 等增强器标签
	private boolean isAdviceNode(Node aNode, ParserContext parserContext) {
		if (!(aNode instanceof Element)) {
			return false;
		}
		else {
			String name = parserContext.getDelegate().getLocalName(aNode);
			return (BEFORE.equals(name) || AFTER.equals(name) || AFTER_RETURNING_ELEMENT.equals(name) ||
					AFTER_THROWING_ELEMENT.equals(name) || AROUND.equals(name));
		}
	}
}
```

### 2.3 advice 标签解析

具体解析 before, after 等增强器标签。

```xml
<aop:before method="printTime" pointcut-ref="addAllMethod"/>
```

方法主要做了三件事：

1. 根据织入方式（before、after这些）创建RootBeanDefinition，名为adviceDef即advice定义
2. 将上一步创建的RootBeanDefinition 写入一个新的RootBeanDefinition，构造一个新的对象，名为advisorDefinition，即advisor定义
3. 将advisorDefinition注册到DefaultListableBeanFactory中

```java
// org.springframework.aop.config.ConfigBeanDefinitionParser
class ConfigBeanDefinitionParser implements BeanDefinitionParser {

    private AbstractBeanDefinition parseAdvice(
			String aspectName, int order, Element aspectElement, Element adviceElement, ParserContext parserContext,
			List<BeanDefinition> beanDefinitions, List<BeanReference> beanReferences) {

		try {
			this.parseState.push(new AdviceEntry(parserContext.getDelegate().getLocalName(adviceElement)));

			// 封装method 属性
			RootBeanDefinition methodDefinition = new RootBeanDefinition(MethodLocatingFactoryBean.class);
			methodDefinition.getPropertyValues().add("targetBeanName", aspectName);
			methodDefinition.getPropertyValues().add("methodName", adviceElement.getAttribute("method"));
			methodDefinition.setSynthetic(true);

			// create instance factory definition
			RootBeanDefinition aspectFactoryDef =
					new RootBeanDefinition(SimpleBeanFactoryAwareAspectInstanceFactory.class);
			aspectFactoryDef.getPropertyValues().add("aspectBeanName", aspectName);
			aspectFactoryDef.setSynthetic(true);

			// 封装切入点属性
			AbstractBeanDefinition adviceDef = createAdviceDefinition(
					adviceElement, parserContext, aspectName, order, methodDefinition, aspectFactoryDef,
					beanDefinitions, beanReferences);

			// 创建 advisor bean 定义
			RootBeanDefinition advisorDefinition = new RootBeanDefinition(AspectJPointcutAdvisor.class);
			advisorDefinition.setSource(parserContext.extractSource(adviceElement));
			advisorDefinition.getConstructorArgumentValues().addGenericArgumentValue(adviceDef);
			// ...

			// advisor bean 定义，注册到IOC 容器
			parserContext.getReaderContext().registerWithGeneratedName(advisorDefinition);

			return advisorDefinition;
		}
		finally {
			this.parseState.pop();
		}
	}

	// 创建 advice bean 定义
    private AbstractBeanDefinition createAdviceDefinition(
			Element adviceElement, ParserContext parserContext, String aspectName, int order,
			RootBeanDefinition methodDef, RootBeanDefinition aspectFactoryDef,
			List<BeanDefinition> beanDefinitions, List<BeanReference> beanReferences) {

		// 创建 advice bean 定义
		RootBeanDefinition adviceDefinition = new RootBeanDefinition(getAdviceClass(adviceElement, parserContext));
		adviceDefinition.setSource(parserContext.extractSource(adviceElement));

		adviceDefinition.getPropertyValues().add(ASPECT_NAME_PROPERTY, aspectName);
		adviceDefinition.getPropertyValues().add(DECLARATION_ORDER_PROPERTY, order);

		// ...
		ConstructorArgumentValues cav = adviceDefinition.getConstructorArgumentValues();
		cav.addIndexedArgumentValue(METHOD_INDEX, methodDef);

		// 取 pointcut-ref 属性
		Object pointcut = parsePointcutProperty(adviceElement, parserContext);
		if (pointcut instanceof BeanDefinition) {
			// ...
		}
		else if (pointcut instanceof String) {
			RuntimeBeanReference pointcutRef = new RuntimeBeanReference((String) pointcut);
			cav.addIndexedArgumentValue(POINTCUT_INDEX, pointcutRef);
			beanReferences.add(pointcutRef);
		}

		cav.addIndexedArgumentValue(ASPECT_INSTANCE_FACTORY_INDEX, aspectFactoryDef);

		return adviceDefinition;
	}

	// 根据具体advice ，返回不同实现类
    private Class<?> getAdviceClass(Element adviceElement, ParserContext parserContext) {
		String elementName = parserContext.getDelegate().getLocalName(adviceElement);
		if (BEFORE.equals(elementName)) {
			return AspectJMethodBeforeAdvice.class;
		}
		else if (AFTER.equals(elementName)) {
			return AspectJAfterAdvice.class;
		}
		else if (AFTER_RETURNING_ELEMENT.equals(elementName)) {
			return AspectJAfterReturningAdvice.class;
		}
		else if (AFTER_THROWING_ELEMENT.equals(elementName)) {
			return AspectJAfterThrowingAdvice.class;
		}
		else if (AROUND.equals(elementName)) {
			return AspectJAroundAdvice.class;
		}
		else {
			throw new IllegalArgumentException("Unknown advice kind [" + elementName + "].");
		}
	}

	@Nullable
	private Object parsePointcutProperty(Element element, ParserContext parserContext) {
		if (element.hasAttribute(POINTCUT) && element.hasAttribute(POINTCUT_REF)) {
			// ...
		}
		else if (element.hasAttribute(POINTCUT_REF)) {
			String pointcutRef = element.getAttribute(POINTCUT_REF);
			
			// ...
			return pointcutRef;
		}
		// ...
	}
}

// org.springframework.beans.factory.xml.XmlReaderContext
public class XmlReaderContext extends ReaderContext {
	// 将bean 定义，注册到IOC 容器中
	public String registerWithGeneratedName(BeanDefinition beanDefinition) {
		// 获取bean 名称
		String generatedName = generateBeanName(beanDefinition);
		getRegistry().registerBeanDefinition(generatedName, beanDefinition);
		return generatedName;
	}
}
```

### 2.4 pointcut 标签解析

```xml
<aop:pointcut id="addAllMethod" expression="execution(* com.hello.Dao.*(..))"/>
```

解析切入点配置:

```java
// org.springframework.aop.config.ConfigBeanDefinitionParser
class ConfigBeanDefinitionParser implements BeanDefinitionParser {
	private AbstractBeanDefinition parsePointcut(Element pointcutElement, ParserContext parserContext) {
		String id = pointcutElement.getAttribute(ID);
		String expression = pointcutElement.getAttribute(EXPRESSION);

		AbstractBeanDefinition pointcutDefinition = null;

		try {
			
			// 创建 Pointcut bean 定义
			pointcutDefinition = createPointcutDefinition(expression);
			pointcutDefinition.setSource(parserContext.extractSource(pointcutElement));

			String pointcutBeanName = id;
			if (StringUtils.hasText(pointcutBeanName)) {
				// 将bean 定义注册到 IOC 容器
				parserContext.getRegistry().registerBeanDefinition(pointcutBeanName, pointcutDefinition);
			}
			else {
				pointcutBeanName = parserContext.getReaderContext().registerWithGeneratedName(pointcutDefinition);
			}

			// ...
		}
		finally {
			this.parseState.pop();
		}

		return pointcutDefinition;
	}

    protected AbstractBeanDefinition createPointcutDefinition(String expression) {
		RootBeanDefinition beanDefinition = new RootBeanDefinition(AspectJExpressionPointcut.class);
		beanDefinition.setScope(BeanDefinition.SCOPE_PROTOTYPE);
		beanDefinition.setSynthetic(true);
		beanDefinition.getPropertyValues().add(EXPRESSION, expression);
		return beanDefinition;
	}
}
```

## 参考资料

- org.springframework.aop.config.ConfigBeanDefinitionParser
- [【Spring源码分析】AOP源码解析（上篇）](https://www.cnblogs.com/xrq730/p/6753160.html)
