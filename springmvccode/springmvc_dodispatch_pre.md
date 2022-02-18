FrameworkServlet
- void doGet (HttpServletRequest request, HttpServletResponse response)
- void processRequest (HttpServletRequest request, HttpServletResponse response)

DispatcherServlet
- void doService (HttpServletRequest request, HttpServletResponse response)
- void doDispatch (HttpServletRequest request, HttpServletResponse response)
- HandlerExecutionChain getHandler (HttpServletRequest request)
- HandlerAdapter getHandlerAdapter (Object handler)
- void processDispatchResult (HttpServletRequest request, HttpServletResponse response, @Nullable HandlerExecutionChain mappedHandler, ......)
- void render (ModelAndView mv, HttpServletRequest request, HttpServletResponse response)

AbstractHandlerMapping
+ HandlerExecutionChain getHandler (HttpServletRequest request)
- HandlerExecutionChain getHandlerExecutionChain (Object handler, HttpServletRequest request)

AbstractHandlerMethodMapping
- HandlerMethod getHandlerInternal (HttpServletRequest request)
- HandlerMethod lookupHandlerMethod (String lookupPath, HttpServletRequest request)

AbstractHandlerMethodAdapter
+ boolean supports (Object handler)
+ ModelAndView handle (HttpServletRequest request, HttpServletResponse response, Object handler)

RequestMappingHandlerAdapter
- ModelAndView handleInternal (HttpServletRequest request, HttpServletResponse response, HandlerMethod handlerMethod)
- ModelAndView invokeHandlerMethod (HttpServletRequest request, HttpServletResponse response, HandlerMethod handlerMethod)
- ModelAndView getModelAndView (ModelAndViewContainer mavContainer, ModelFactory modelFactory, NativeWebRequest webRequest)

ServletInvocableHandlerMethod
+ void invokeAndHandle (ServletWebRequest webRequest, ModelAndViewContainer mavContainer, Object... providedArgs)

InvocableHandlerMethod
+ Object invokeForRequest (NativeWebRequest request, @Nullable ModelAndViewContainer mavContainer, Object... providedArgs)

AbstractView
+ void render (@Nullable Map<String, ?> model, HttpServletRequest request, HttpServletResponse response)

InternalResourceView
- void renderMergedOutputModel (Map<String, Object> model, HttpServletRequest request, HttpServletResponse response)
- RequestDispatcher getRequestDispatcher (HttpServletRequest request, String path)



# DispatcherServlet 的逻辑处理
## 1. 请求上下文
```java
void doGet (HttpServletRequest request, HttpServletResponse response)
```

```java
processRequest(request, response);
```

```java
void processRequest (HttpServletRequest request, HttpServletResponse response)
```

```java
doService(request, response);
```

```java
void doService (HttpServletRequest request, HttpServletResponse response)
```

```java
doDispatch(request, response);
```

## 2. 请求分发 doDispatch
```java
void doDispatch (HttpServletRequest request, HttpServletResponse response)
```

```java
mappedHandler = getHandler(processedRequest);
```

```java
HandlerAdapter ha = getHandlerAdapter(mappedHandler.getHandler());
```

```java
if (!mappedHandler.applyPreHandle(processedRequest, response)) {
    return;
}
```

```java
mv = ha.handle(processedRequest, response, mappedHandler.getHandler());
```

```java
applyDefaultViewName(processedRequest, mv);
mappedHandler.applyPostHandle(processedRequest, response, mv);
```

```java
processDispatchResult(processedRequest, response, mappedHandler, mv, dispatchException);
```

```java
HandlerExecutionChain getHandler (HttpServletRequest request)
```

```java
for (HandlerMapping mapping : this.handlerMappings) {
    HandlerExecutionChain handler = mapping.getHandler(request);
    if (handler != null) {
        return handler;
    }
}
```

```java
HandlerExecutionChain getHandler (HttpServletRequest request)
```

```java
Object handler = getHandlerInternal(request);
```

```java
HandlerExecutionChain executionChain = getHandlerExecutionChain(handler, request);
return executionChain;
```

```java
HandlerExecutionChain getHandlerExecutionChain (Object handler, HttpServletRequest request)
```

```java
String lookupPath = this.urlPathHelper.getLookupPathForRequest(request, LOOKUP_PATH);
for (HandlerInterceptor interceptor : this.adaptedInterceptors) {
    if (interceptor instanceof MappedInterceptor) {
        MappedInterceptor mappedInterceptor = (MappedInterceptor) interceptor;
        if (mappedInterceptor.matches(lookupPath, this.pathMatcher)) {
            chain.addInterceptor(mappedInterceptor.getInterceptor());
        }
    }
}
```

```java
HandlerMethod getHandlerInternal (HttpServletRequest request)
```

```java
String lookupPath = getUrlPathHelper().getLookupPathForRequest(request);
HandlerMethod handlerMethod = lookupHandlerMethod(lookupPath, request);
return (handlerMethod != null ? handlerMethod.createWithResolvedBean() : null);
```

```java
HandlerMethod lookupHandlerMethod (String lookupPath, HttpServletRequest request)
```

```java
List<T> directPathMatches = this.mappingRegistry.getMappingsByUrl(lookupPath);
if (!matches.isEmpty()) {
    Match bestMatch = matches.get(0);
    return bestMatch.handlerMethod;
}
```

```java
HandlerAdapter getHandlerAdapter (Object handler)
```

```java
for (HandlerAdapter adapter : this.handlerAdapters) {
    if (adapter.supports(handler)) {
        return adapter;
    }
}
```

```java
boolean supports (Object handler)
```

```java
return (handler instanceof HandlerMethod && supportsInternal((HandlerMethod) handler));
```

## 3. 请求处理
```java
ModelAndView handle (HttpServletRequest request, HttpServletResponse response, Object handler)
```

```java
return handleInternal(request, response, (HandlerMethod) handler);
```

```java
ModelAndView handleInternal (HttpServletRequest request, HttpServletResponse response, HandlerMethod handlerMethod)
```

```java
mav = invokeHandlerMethod(request, response, handlerMethod);
return mav;
```

```java
ModelAndView invokeHandlerMethod (HttpServletRequest request, HttpServletResponse response, HandlerMethod handlerMethod)
```

```java
invocableMethod.invokeAndHandle(webRequest, mavContainer);
```

```java
return getModelAndView(mavContainer, modelFactory, webRequest);
```

```java
ModelAndView getModelAndView (ModelAndViewContainer mavContainer, ModelFactory modelFactory, NativeWebRequest webRequest)
```

```java
ModelMap model = mavContainer.getModel();
ModelAndView mav = new ModelAndView(mavContainer.getViewName(), model, mavContainer.getStatus());
return mav;
```

```java
void invokeAndHandle (ServletWebRequest webRequest, ModelAndViewContainer mavContainer, Object... providedArgs)
```

```java
Object returnValue = invokeForRequest(webRequest, mavContainer, providedArgs);
this.returnValueHandlers.handleReturnValue(returnValue, getReturnValueType(returnValue), mavContainer, webRequest);
```

```java
Object invokeForRequest (NativeWebRequest request, @Nullable ModelAndViewContainer mavContainer, Object... providedArgs)
```

```java
return doInvoke(args);
```

## 4. 视图渲染
```java
void processDispatchResult (HttpServletRequest request, HttpServletResponse response, @Nullable HandlerExecutionChain mappedHandler, ......)
```

```java
render(mv, request, response);
```

```java
void render (ModelAndView mv, HttpServletRequest request, HttpServletResponse response)
```

```java
String viewName = mv.getViewName();
view = resolveViewName(viewName, mv.getModelInternal(), locale, request);
view.render(mv.getModelInternal(), request, response);
```

```java
void render (@Nullable Map<String, ?> model, HttpServletRequest request, HttpServletResponse response)
```

```java
renderMergedOutputModel(mergedModel, getRequestToExpose(request), response);
```

```java
void renderMergedOutputModel (Map<String, Object> model, HttpServletRequest request, HttpServletResponse response)
```

```java
RequestDispatcher rd = getRequestDispatcher(request, dispatcherPath);
rd.forward(request, response);
```
