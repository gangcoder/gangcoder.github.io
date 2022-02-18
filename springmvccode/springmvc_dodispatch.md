<!-- ---
title: DispatcherServlet 的逻辑处理
date: 2022-01-23 15:51:17
category: java100, springmvc, code
--- -->

# DispatcherServlet 的逻辑处理


## 1. 请求上下文

doGet 、doPost 调用 processRequest 方法进行处理，最终是委托给子类 DispatcherServlet 的 doServie() 方法。

```java
public abstract class FrameworkServlet extends HttpServletBean implements ApplicationContextAware {
	@Override
	protected final void doGet(HttpServletRequest request, HttpServletResponse response)
			throws ServletException, IOException {

		processRequest(request, response);
	}

	protected final void processRequest(HttpServletRequest request, HttpServletResponse response)
			throws ServletException, IOException {
        // ...

        // 进入请求处理过程
        doService(request, response);
    }
}

public class DispatcherServlet extends FrameworkServlet {
	@Override
	protected void doService(HttpServletRequest request, HttpServletResponse response) throws Exception {
        // ...
        doDispatch(request, response);
    }
}
```


## 2. 请求分发 doDispatch

```java
public class DispatcherServlet extends FrameworkServlet {

	protected void doDispatch(HttpServletRequest request, HttpServletResponse response) throws Exception {
        // ...
        try {
            // ...

            // 根据 request 信息寻找对应的 Handler
            mappedHandler = getHandler(processedRequest);
            
            // 根据当前的 handler 找到对应的 HandlerAdapter 适配器
            HandlerAdapter ha = getHandlerAdapter(mappedHandler.getHandler());

            // 拦截器的 preHandler 方法的调用
            if (!mappedHandler.applyPreHandle(processedRequest, response)) {
                return;
            }

            // 真正激活 handler 进行处理，并返回视图
            mv = ha.handle(processedRequest, response, mappedHandler.getHandler());

            // 视图名称转换
            applyDefaultViewName(processedRequest, mv);

            // 应用所有拦截器的 postHandle 方法
            mappedHandler.applyPostHandle(processedRequest, response, mv);
        }
        catch (Exception ex) {
            // ...
        }

        // 处理分发的结果，视图渲染
        processDispatchResult(processedRequest, response, mappedHandler, mv, dispatchException);

    }

}
```

根据 request 信息寻找对应的 Handler。

```java
public class DispatcherServlet extends FrameworkServlet {
	@Nullable
	protected HandlerExecutionChain getHandler(HttpServletRequest request) throws Exception {
		if (this.handlerMappings != null) {
			for (HandlerMapping mapping : this.handlerMappings) {
				HandlerExecutionChain handler = mapping.getHandler(request);
				if (handler != null) {
					return handler;
				}
			}
		}
		return null;
	}
}

public abstract class AbstractHandlerMapping extends WebApplicationObjectSupport
		implements HandlerMapping, Ordered, BeanNameAware {
	public final HandlerExecutionChain getHandler(HttpServletRequest request) throws Exception {
        // 根据 Request 获取对应的 handler
		Object handler = getHandlerInternal(request);

        // ...

        // 将配置中的对应拦截器加入到执行链中，以保证这些拦截器可以有效地作用于目标对象
        HandlerExecutionChain executionChain = getHandlerExecutionChain(handler, request);

        return executionChain;
    }

	protected HandlerExecutionChain getHandlerExecutionChain(Object handler, HttpServletRequest request) {
		HandlerExecutionChain chain = (handler instanceof HandlerExecutionChain ?
				(HandlerExecutionChain) handler : new HandlerExecutionChain(handler));

		String lookupPath = this.urlPathHelper.getLookupPathForRequest(request, LOOKUP_PATH);
        // 寻找匹配的拦截器
		for (HandlerInterceptor interceptor : this.adaptedInterceptors) {
			if (interceptor instanceof MappedInterceptor) {
				MappedInterceptor mappedInterceptor = (MappedInterceptor) interceptor;
				if (mappedInterceptor.matches(lookupPath, this.pathMatcher)) {
					chain.addInterceptor(mappedInterceptor.getInterceptor());
				}
			}
			// ...
		}
		return chain;
	}
}

public abstract class AbstractHandlerMethodMapping<T> extends AbstractHandlerMapping implements InitializingBean {
	@Override
	protected HandlerMethod getHandlerInternal(HttpServletRequest request) throws Exception {
		String lookupPath = getUrlPathHelper().getLookupPathForRequest(request);
		
        // ...
		HandlerMethod handlerMethod = lookupHandlerMethod(lookupPath, request);
        return (handlerMethod != null ? handlerMethod.createWithResolvedBean() : null);

        // ...
    }

	@Nullable
	protected HandlerMethod lookupHandlerMethod(String lookupPath, HttpServletRequest request) throws Exception {
		List<T> directPathMatches = this.mappingRegistry.getMappingsByUrl(lookupPath);
		
        // ...

		if (!matches.isEmpty()) {
			Match bestMatch = matches.get(0);
			
            // ...
			return bestMatch.handlerMethod;
		}
	}
}
```

寻找适配器 HandlerAdapter。


```java
public class DispatcherServlet extends FrameworkServlet {
	protected HandlerAdapter getHandlerAdapter(Object handler) throws ServletException {
		if (this.handlerAdapters != null) {
			for (HandlerAdapter adapter : this.handlerAdapters) {
				if (adapter.supports(handler)) {
					return adapter;
				}
			}
		}
		// ...
	}
}

public abstract class AbstractHandlerMethodAdapter extends WebContentGenerator implements HandlerAdapter, Ordered {
	@Override
	public final boolean supports(Object handler) {
		return (handler instanceof HandlerMethod && supportsInternal((HandlerMethod) handler));
	}
}
```

## 3. 请求处理

```java
public abstract class AbstractHandlerMethodAdapter extends WebContentGenerator implements HandlerAdapter, Ordered {
	@Override
	@Nullable
	public final ModelAndView handle(HttpServletRequest request, HttpServletResponse response, Object handler)
			throws Exception {

		return handleInternal(request, response, (HandlerMethod) handler);
	}
}

public class RequestMappingHandlerAdapter extends AbstractHandlerMethodAdapter
		implements BeanFactoryAware, InitializingBean {
	@Override
	protected ModelAndView handleInternal(HttpServletRequest request,
			HttpServletResponse response, HandlerMethod handlerMethod) throws Exception {
        // ...
        // 执行适配中真正的方法
        mav = invokeHandlerMethod(request, response, handlerMethod);

        return mav;
    }

	@Nullable
	protected ModelAndView invokeHandlerMethod(HttpServletRequest request,
			HttpServletResponse response, HandlerMethod handlerMethod) throws Exception {
        // ...

        // 调用方法
        invocableMethod.invokeAndHandle(webRequest, mavContainer);
        
        // ...

        // 获取模型数据和视图
        return getModelAndView(mavContainer, modelFactory, webRequest);
    }

	@Nullable
	private ModelAndView getModelAndView(ModelAndViewContainer mavContainer,
			ModelFactory modelFactory, NativeWebRequest webRequest) throws Exception {
        // ...

        // 模型数据和视图
		ModelMap model = mavContainer.getModel();
		ModelAndView mav = new ModelAndView(mavContainer.getViewName(), model, mavContainer.getStatus());
		
        // ...
		return mav;
	}
}

public class ServletInvocableHandlerMethod extends InvocableHandlerMethod {
	public void invokeAndHandle(ServletWebRequest webRequest, ModelAndViewContainer mavContainer,
			Object... providedArgs) throws Exception {
        
        // 请求处理
		Object returnValue = invokeForRequest(webRequest, mavContainer, providedArgs);

        // ...

        // 返回视图解析
        this.returnValueHandlers.handleReturnValue(
					returnValue, getReturnValueType(returnValue), mavContainer, webRequest);
    }
}

public class InvocableHandlerMethod extends HandlerMethod {
	@Nullable
	public Object invokeForRequest(NativeWebRequest request, @Nullable ModelAndViewContainer mavContainer,
			Object... providedArgs) throws Exception {

		// ...
		return doInvoke(args);
	}
}
```


## 4. 视图渲染

渲染调用结果。

```java
public class DispatcherServlet extends FrameworkServlet {
	private void processDispatchResult(HttpServletRequest request, HttpServletResponse response,
			@Nullable HandlerExecutionChain mappedHandler, @Nullable ModelAndView mv,
			@Nullable Exception exception) throws Exception {
        // ...

        render(mv, request, response);
	}

	protected void render(ModelAndView mv, HttpServletRequest request, HttpServletResponse response) throws Exception {
		String viewName = mv.getViewName();
		if (viewName != null) {
			view = resolveViewName(viewName, mv.getModelInternal(), locale, request);
		}

        // ...
        view.render(mv.getModelInternal(), request, response);
    }
}

public abstract class AbstractView extends WebApplicationObjectSupport implements View, BeanNameAware {
	@Override
	public void render(@Nullable Map<String, ?> model, HttpServletRequest request,
			HttpServletResponse response) throws Exception {
        // ...

		Map<String, Object> mergedModel = createMergedOutputModel(model, request, response);
		prepareResponse(request, response);
		renderMergedOutputModel(mergedModel, getRequestToExpose(request), response);
	}
}

public class InternalResourceView extends AbstractUrlBasedView {
	@Override
	protected void renderMergedOutputModel(
			Map<String, Object> model, HttpServletRequest request, HttpServletResponse response) throws Exception {
        // ...

        RequestDispatcher rd = getRequestDispatcher(request, dispatcherPath);

        // ...
        // tomcat 完成最终渲染
        rd.forward(request, response);
    }

	@Nullable
	protected RequestDispatcher getRequestDispatcher(HttpServletRequest request, String path) {
        // tomcat 获取 RequestDispatcher
		return request.getRequestDispatcher(path);
	}
}
```


## 参考资料

- [Spring 源码学习(十) Spring mvc](http://www.justdojava.com/2019/07/21/spring-analysis-note-10)

