---
title: SpringMVC源码解析(二) - 怎样找到处理Http请求的Method
date: 2018-11-17 12:02:24
tags: [spring, 源码解析, springMVC]
categories: [spring源码解析]
---

# 1 获取 HandlerExecutionChain

DispatchServlet 的 doDispatch 方法中调用了 getHandler 方法获取了执行请求的 HandlerExecutionChain。
HandlerExecutionChain 包含了拦截器已经处理该请求的 handler 等信息。

来看一下 doDispatch 的源码片段

```java
protected void doDispatch(HttpServletRequest request, HttpServletResponse response) throws Exception {
  ...
  ...
        processedRequest = checkMultipart(request);
        multipartRequestParsed = (processedRequest != request);

        // Determine handler for the current request.
        mappedHandler = getHandler(processedRequest);
        if (mappedHandler == null) {
          noHandlerFound(processedRequest, response);
          return;
        }
  ...
  ...

```

## 1.1 获取 HandlerExecutionChain

在来看一下获取 HandlerExecutionChain 实例 mappedHandler 的方法 getHandler

```java
  @Nullable
  protected HandlerExecutionChain getHandler(HttpServletRequest request) throws Exception {
    if (this.handlerMappings != null) {
      for (HandlerMapping hm : this.handlerMappings) {
        if (logger.isTraceEnabled()) {
          logger.trace(
              "Testing handler map [" + hm + "] in DispatcherServlet with name '" + getServletName() + "'");
        }
        HandlerExecutionChain handler = hm.getHandler(request);
        if (handler != null) {
          return handler;
        }
      }
    }
    return null;
  }
```

这个方法，for 循环了 this.handlerMappings 列表，当列表中的 HandlerMapping 元素 hm 能取到 handler 则立即返回。
那么这个类属性列表 this.handlerMappings 如何初始化的呢？
我们来看一下它的初始化方法 DispatcherServlet.initHandlerMappings

```java
  private void initHandlerMappings(ApplicationContext context) {
    this.handlerMappings = null;

    if (this.detectAllHandlerMappings) {
      // Find all HandlerMappings in the ApplicationContext, including ancestor contexts.
      Map<String, HandlerMapping> matchingBeans =
          BeanFactoryUtils.beansOfTypeIncludingAncestors(context, HandlerMapping.class, true, false);
      if (!matchingBeans.isEmpty()) {
        this.handlerMappings = new ArrayList<>(matchingBeans.values());
        // We keep HandlerMappings in sorted order.
        AnnotationAwareOrderComparator.sort(this.handlerMappings);
      }
    }
    else {
      try {
        HandlerMapping hm = context.getBean(HANDLER_MAPPING_BEAN_NAME, HandlerMapping.class);
        this.handlerMappings = Collections.singletonList(hm);
      }
      catch (NoSuchBeanDefinitionException ex) {
        // Ignore, we'll add a default HandlerMapping later.
      }
    }

    // Ensure we have at least one HandlerMapping, by registering
    // a default HandlerMapping if no other mappings are found.
    if (this.handlerMappings == null) {
      this.handlerMappings = getDefaultStrategies(context, HandlerMapping.class);
      if (logger.isDebugEnabled()) {
        logger.debug("No HandlerMappings found in servlet '" + getServletName() + "': using default");
      }
    }
  }
```

因为 this.detectAllHandlerMappings 默认为 true 所以初始化方法从 ApplicationContext 里获取了所有继承
HandlerMapping 接口的 bean，并把它加入到了 handlerMappings 中。
SpringBoot 为我们默认注入了一下几个 geHandlerMapping 的实现类

- SimpleUrlHandlerMapping
- RequestMappingHandlerMapping
- BeanNameUrlHanderMapping
- WebMvcConfigurationSupport
- WelcomePageHanderMapping

限于篇幅，这里将重点分析我们常用的 RequestMappingHandlerMapping。

Springbooot 框架中 ，RequestMappingHandlerMapping 是在哪里被注入的呢？
是在 spring-boot-autoconfigure 包中的 WebMvcAutoConfiguration 里注入的。

```java
@Configuration
@ConditionalOnWebApplication(type = Type.SERVLET)
@ConditionalOnClass({ Servlet.class, DispatcherServlet.class, WebMvcConfigurer.class })
@ConditionalOnMissingBean(WebMvcConfigurationSupport.class)
@AutoConfigureOrder(Ordered.HIGHEST_PRECEDENCE + 10)
@AutoConfigureAfter({ DispatcherServletAutoConfiguration.class,
    ValidationAutoConfiguration.class })
public class WebMvcAutoConfiguration {
  ...
  ...

    @Bean
    @Primary
    @Override
    public RequestMappingHandlerMapping requestMappingHandlerMapping() {
      // Must be @Primary for MvcUriComponentsBuilder to work
      return super.requestMappingHandlerMapping();
    }

  ...
  ...
}
```

## 1.2 RequestMappingHandlerMapping,构建所有@RequestMapping 接口的注册表

RequestMappingHandlerMapping 实例中构建了一个 所有@RequestMapping 接口的注册表，我们来看一下它的初始化构建过程。

### 1.2.1 RequestMappingHandlerMapping 类图

{% asset_img RequestMappingHandlerMapping_uml_class.jpg RequestMappingHandlerMapping类图 %}

### 1.2.2 RequestMappingHandlerMapping 初始化

RequestMappingHandlerMapping 的父类 AbstractHandlerMethodMapping 实现了 InitializedBean 接口。
在 spring 初始化 bean 的时候，如果 bean 实现了 InitializingBean 接口，会自动调用 afterPropertiesSet 方法。

来看 RequestMappingHandlerMapping 的 afterPropertiesSet 方法：

```java
public class RequestMappingHandlerMapping extends RequestMappingInfoHandlerMapping
    implements MatchableHandlerMapping, EmbeddedValueResolverAware {
      ...
      ...
  @Override
  public void afterPropertiesSet() {
    this.config = new RequestMappingInfo.BuilderConfiguration();
    this.config.setUrlPathHelper(getUrlPathHelper());
    this.config.setPathMatcher(getPathMatcher());
    this.config.setSuffixPatternMatch(this.useSuffixPatternMatch);
    this.config.setTrailingSlashMatch(this.useTrailingSlashMatch);
    this.config.setRegisteredSuffixPatternMatch(this.useRegisteredSuffixPatternMatch);
    this.config.setContentNegotiationManager(getContentNegotiationManager());

    super.afterPropertiesSet();
  }
}
```

在做了一些配置之后调用了祖先 AbstractHandlerMethodMapping 的 afterPropertiesSet() (父类 RequestMappingInfoHandlerMapping 没有相应实现)

```java
public abstract class AbstractHandlerMethodMapping<T> extends AbstractHandlerMapping implements InitializingBean {
  ...
  ...
  @Override
  public void afterPropertiesSet() {
    initHandlerMethods();
  }
  /**
   * Scan beans in the ApplicationContext, detect and register handler methods.
   * @see #isHandler(Class)
   * @see #getMappingForMethod(Method, Class)
   * @see #handlerMethodsInitialized(Map)
   */
  protected void initHandlerMethods() {
    if (logger.isDebugEnabled()) {
      logger.debug("Looking for request mappings in application context: " + getApplicationContext());
    }
    String[] beanNames = (this.detectHandlerMethodsInAncestorContexts ?
        BeanFactoryUtils.beanNamesForTypeIncludingAncestors(obtainApplicationContext(), Object.class) :
        obtainApplicationContext().getBeanNamesForType(Object.class));

    for (String beanName : beanNames) {
      if (!beanName.startsWith(SCOPED_TARGET_NAME_PREFIX)) {
        Class<?> beanType = null;
        try {
          beanType = obtainApplicationContext().getType(beanName);
        }
        catch (Throwable ex) {
          // An unresolvable bean type, probably from a lazy bean - let's ignore it.
          if (logger.isDebugEnabled()) {
            logger.debug("Could not resolve target class for bean with name '" + beanName + "'", ex);
          }
        }
        if (beanType != null && isHandler(beanType)) {
          detectHandlerMethods(beanName);
        }
      }
    }
    handlerMethodsInitialized(getHandlerMethods());
  }
}
```

```java
public class RequestMappingHandlerMapping extends RequestMappingInfoHandlerMapping
    implements MatchableHandlerMapping, EmbeddedValueResolverAware {
  ...
  ...
  @Override
  protected boolean isHandler(Class<?> beanType) {
    return (AnnotatedElementUtils.hasAnnotation(beanType, Controller.class) ||
        AnnotatedElementUtils.hasAnnotation(beanType, RequestMapping.class));
  }
  ...
  ...
}
```

这里调用了 initHandlerMethods 来查找所有 Handler 的方法。

1. 第一步把 ApplicationContext 里所有的 bean 的名称全部去出来。
2. 根据 bean 的名称获取 bean 的类型 beanType
3. 调用 isHandler 判断是否是 Handler，
4. RequestMappingHandlerMapping 的 isHanlder 方法，根据类型是否有@Controller 或
   @RequestMapping 注解判断是否是 Handler
5. 如果是 Handler 调用 detectHandlerMethods 注册所有的 handler 方法。

我们来看一下 detectHandlerMethods 方法

```java
public abstract class AbstractHandlerMethodMapping<T> extends AbstractHandlerMapping implements InitializingBean {
  ...
  ...
  protected void detectHandlerMethods(final Object handler) {
    Class<?> handlerType = (handler instanceof String ?
        obtainApplicationContext().getType((String) handler) : handler.getClass());

    if (handlerType != null) {
      final Class<?> userType = ClassUtils.getUserClass(handlerType);
      Map<Method, T> methods = MethodIntrospector.selectMethods(userType,
          (MethodIntrospector.MetadataLookup<T>) method -> {
            try {
              return getMappingForMethod(method, userType);
            }
            catch (Throwable ex) {
              throw new IllegalStateException("Invalid mapping on handler class [" +
                  userType.getName() + "]: " + method, ex);
            }
          });
      if (logger.isDebugEnabled()) {
        logger.debug(methods.size() + " request handler methods found on " + userType + ": " + methods);
      }
      methods.forEach((method, mapping) -> {
        Method invocableMethod = AopUtils.selectInvocableMethod(method, userType);
        registerHandlerMethod(handler, invocableMethod, mapping);
      });
    }
  }

  /**
   * Register a handler method and its unique mapping. Invoked at startup for
   * each detected handler method.
   * @param handler the bean name of the handler or the handler instance
   * @param method the method to register
   * @param mapping the mapping conditions associated with the handler method
   * @throws IllegalStateException if another method was already registered
   * under the same mapping
   */
  protected void registerHandlerMethod(Object handler, Method method, T mapping) {
    this.mappingRegistry.register(mapping, handler, method);
  }

}
```

```java
public class RequestMappingHandlerMapping extends RequestMappingInfoHandlerMapping
    implements MatchableHandlerMapping, EmbeddedValueResolverAware {
  ...
  ...
  @Override
  @Nullable
  protected RequestMappingInfo getMappingForMethod(Method method, Class<?> handlerType) {
    RequestMappingInfo info = createRequestMappingInfo(method);
    if (info != null) {
      RequestMappingInfo typeInfo = createRequestMappingInfo(handlerType);
      if (typeInfo != null) {
        info = typeInfo.combine(info);
      }
    }
    return info;
  }

  /**
   * Delegates to {@link #createRequestMappingInfo(RequestMapping, RequestCondition)},
   * supplying the appropriate custom {@link RequestCondition} depending on whether
   * the supplied {@code annotatedElement} is a class or method.
   * @see #getCustomTypeCondition(Class)
   * @see #getCustomMethodCondition(Method)
   */
  @Nullable
  private RequestMappingInfo createRequestMappingInfo(AnnotatedElement element) {
    RequestMapping requestMapping = AnnotatedElementUtils.findMergedAnnotation(element, RequestMapping.class);
    RequestCondition<?> condition = (element instanceof Class ?
        getCustomTypeCondition((Class<?>) element) : getCustomMethodCondition((Method) element));
    return (requestMapping != null ? createRequestMappingInfo(requestMapping, condition) : null);
  }



}
```

```java
    public void register(T mapping, Object handler, Method method) {
      this.readWriteLock.writeLock().lock();
      try {
        HandlerMethod handlerMethod = createHandlerMethod(handler, method);
        assertUniqueMethodMapping(handlerMethod, mapping);

        if (logger.isInfoEnabled()) {
          logger.info("Mapped \"" + mapping + "\" onto " + handlerMethod);
        }
        this.mappingLookup.put(mapping, handlerMethod);

        List<String> directUrls = getDirectUrls(mapping);
        for (String url : directUrls) {
          this.urlLookup.add(url, mapping);
        }

        String name = null;
        if (getNamingStrategy() != null) {
          name = getNamingStrategy().getName(handlerMethod, mapping);
          addMappingName(name, handlerMethod);
        }

        CorsConfiguration corsConfig = initCorsConfiguration(handler, method, mapping);
        if (corsConfig != null) {
          this.corsLookup.put(handlerMethod, corsConfig);
        }

        this.registry.put(mapping, new MappingRegistration<>(mapping, handlerMethod, directUrls, name));
      }
      finally {
        this.readWriteLock.writeLock().unlock();
      }
    }

```

detectHandlerMethods

1. 先调用了 MethodIntrospector.selectMethods。MethodIntrospector.selectMethods 的第二参数传入了一个 filter 函数。当返回不是 null 的时候，将会把返回结果保存在 Map<Method,T>中。
2. filter 中调用了 getMappingForMethod 方法,传入 method 和 该方法所在类 handlerType
3. 调用 createRequestMappingInfo 传入 method，如果这个方法被@RequestMapping 注解修饰，就会返回一个 RequestMappingInfo 实例，RequestMappingInfo 实例记录接口 method 处理 HttpRequest 的所有信息，包括路径，接口允许的方法，CORS 信息，已经各种限制条件。
4. 如果返回实例不为 null，再次调用 getMappingForMethod 方法，传入该方法所在类型 handlerType 获得一个基于类的 RequestMappingInfo 实例，
5. 两个实例 merge 得到一个新的 RequestMappingInfo 实例。（请求路径的合并等。。）
6. 得到所有 methods 的 map 以后进行遍历，调用 registerHandlerMethod 进行注册保存。
7. 调用了 this.mappingRegistry.register 进行注册。保存了 mapping(requestMappingInfo)和 handlerMethod 的关系，
   以及 urlString 与 mapping(requestMappingInfo)的关系。

### 1.2.3 初始化总结

在完成了 RequestMappingHandlerMapping 的初始化以后，这个实例中便保存了我们所有@RequestMapping 修饰的接口信息。

## 1.3 RequestMappingHanderMapping 获取 handlerChain

我们回到文章的第一段代码

```java
protected HandlerExecutionChain getHandler(HttpServletRequest request) throws Exception {
    if (this.handlerMappings != null) {
      for (HandlerMapping hm : this.handlerMappings) {
        if (logger.isTraceEnabled()) {
          logger.trace(
              "Testing handler map [" + hm + "] in DispatcherServlet with name '" + getServletName() + "'");
        }
        HandlerExecutionChain handler = hm.getHandler(request);
        if (handler != null) {
          return handler;
        }
      }
    }
    return null;
  }
```

handlerMapping 调用了 getHandler(request)获取 HandlerExecutionChain 实例，那么我们来看一下
RequestMappingHanderMapping 的 getHandler 方法。

```java
public abstract class AbstractHandlerMapping extends WebApplicationObjectSupport implements HandlerMapping, Ordered {
  @Override
  @Nullable
  public final HandlerExecutionChain getHandler(HttpServletRequest request) throws Exception {
    Object handler = getHandlerInternal(request);
    if (handler == null) {
      handler = getDefaultHandler();
    }
    if (handler == null) {
      return null;
    }
    // Bean name or resolved handler?
    if (handler instanceof String) {
      String handlerName = (String) handler;
      handler = obtainApplicationContext().getBean(handlerName);

    HandlerExecutionChain executionChain = getHandlerExecutionChain(handler, request);
    if (CorsUtils.isCorsRequest(request)) {
      CorsConfiguration globalConfig = this.globalCorsConfigSource.getCorsConfiguration(request);
      CorsConfiguration handlerConfig = getCorsConfiguration(handler, request);
      CorsConfiguration config = (globalConfig != null ? globalConfig.combine(handlerConfig) : handlerConfig);
      executionChain = getCorsHandlerExecutionChain(request, executionChain, config);
    }
    return executionChain;
  }
}
```

调用了 getHandlerInternal,
从 mappingRegistry 中获取匹配路径的 mapping，并排序获取最匹配的 handlerMethod

```java
public abstract class AbstractHandlerMethodMapping<T> extends AbstractHandlerMapping implements InitializingBean {
  @Override
  protected HandlerMethod getHandlerInternal(HttpServletRequest request) throws Exception {
    String lookupPath = getUrlPathHelper().getLookupPathForRequest(request);
    if (logger.isDebugEnabled()) {
      logger.debug("Looking up handler method for path " + lookupPath);
    }
    this.mappingRegistry.acquireReadLock();
    try {
      HandlerMethod handlerMethod = lookupHandlerMethod(lookupPath, request);
      if (logger.isDebugEnabled()) {
        if (handlerMethod != null) {
          logger.debug("Returning handler method [" + handlerMethod + "]");
        }
        else {
          logger.debug("Did not find handler method for [" + lookupPath + "]");
        }
      }
      return (handlerMethod != null ? handlerMethod.createWithResolvedBean() : null);
    }
    finally {
      this.mappingRegistry.releaseReadLock();
    }
  }

  protected HandlerMethod lookupHandlerMethod(String lookupPath, HttpServletRequest request) throws Exception {
    List<Match> matches = new ArrayList<>();
    List<T> directPathMatches = this.mappingRegistry.getMappingsByUrl(lookupPath);
    if (directPathMatches != null) {
      addMatchingMappings(directPathMatches, matches, request);
    }
    if (matches.isEmpty()) {
      // No choice but to go through all mappings...
      addMatchingMappings(this.mappingRegistry.getMappings().keySet(), matches, request);
    }

    if (!matches.isEmpty()) {
      Comparator<Match> comparator = new MatchComparator(getMappingComparator(request));
      matches.sort(comparator);
      if (logger.isTraceEnabled()) {
        logger.trace("Found " + matches.size() + " matching mapping(s) for [" + lookupPath + "] : " + matches);
      }
      Match bestMatch = matches.get(0);
      if (matches.size() > 1) {
        if (CorsUtils.isPreFlightRequest(request)) {
          return PREFLIGHT_AMBIGUOUS_MATCH;
        }
        Match secondBestMatch = matches.get(1);
        if (comparator.compare(bestMatch, secondBestMatch) == 0) {
          Method m1 = bestMatch.handlerMethod.getMethod();
          Method m2 = secondBestMatch.handlerMethod.getMethod();
          throw new IllegalStateException("Ambiguous handler methods mapped for HTTP path '" +
              request.getRequestURL() + "': {" + m1 + ", " + m2 + "}");
        }
      }
      handleMatch(bestMatch.mapping, lookupPath, request);
      return bestMatch.handlerMethod;
    }
    else {
      return handleNoMatch(this.mappingRegistry.getMappings().keySet(), lookupPath, request);
    }
  }

  private void addMatchingMappings(Collection<T> mappings, List<Match> matches, HttpServletRequest request) {
    for (T mapping : mappings) {
      T match = getMatchingMapping(mapping, request);
      if (match != null) {
        matches.add(new Match(match, this.mappingRegistry.getMappings().get(mapping)));
      }
    }
  }
}
```

获取到了 handlerMethod 以后调用了 getHandlerExecutionChain 把匹配的 Intercepters 组装成了 HandlerExecutionChain 对象

```java
  protected HandlerExecutionChain getHandlerExecutionChain(Object handler, HttpServletRequest request) {
    HandlerExecutionChain chain = (handler instanceof HandlerExecutionChain ?
        (HandlerExecutionChain) handler : new HandlerExecutionChain(handler));

    String lookupPath = this.urlPathHelper.getLookupPathForRequest(request);
    for (HandlerInterceptor interceptor : this.adaptedInterceptors) {
      if (interceptor instanceof MappedInterceptor) {
        MappedInterceptor mappedInterceptor = (MappedInterceptor) interceptor;
        if (mappedInterceptor.matches(lookupPath, this.pathMatcher)) {
          chain.addInterceptor(mappedInterceptor.getInterceptor());
        }
      }
      else {
        chain.addInterceptor(interceptor);
      }
    }
    return chain;
  }
```

# 2. 获取 HandlerAdapter

回到 doDispatchServlet 的代码片段

```java
        // Determine handler adapter for the current request.
        HandlerAdapter ha = getHandlerAdapter(mappedHandler.getHandler());
```

调用了 getHandlerAdapter 获取 HandlerAdapter 实例 ha。
DispatchServlet 初始化的时候会把 appContext 里所有实现了 HandlerAdapter 的 bean 添加到 this.handlerAdapters 里，
然后通过 for 循环查找支持传入的 handler 的 HanderAdapter。支持 RequestMappingHandlerMapping 的是
RequestMappingHandlerAdapter。

```java
  protected HandlerAdapter getHandlerAdapter(Object handler) throws ServletException {
    if (this.handlerAdapters != null) {
      for (HandlerAdapter ha : this.handlerAdapters) {
        if (logger.isTraceEnabled()) {
          logger.trace("Testing handler adapter [" + ha + "]");
        }
        if (ha.supports(handler)) {
          return ha;
        }
      }
    }
    throw new ServletException("No adapter for handler [" + handler +
        "]: The DispatcherServlet configuration needs to include a HandlerAdapter that supports this handler");
  }
```

RequestMappingHandlerAdapter 的 support 方法，如果 handler 的类型是 HandlerMethod 即支持，RequestMappingHandlerMapping 返回的 handler 类型就是 HandlerMethod

```java
  @Override
  public final boolean supports(Object handler) {
    return (handler instanceof HandlerMethod && supportsInternal((HandlerMethod) handler));
  }

  @Override
  protected boolean supportsInternal(HandlerMethod handlerMethod) {
    return true;
  }
```

# 3. HandlerAdapter 执行 handler 方法。

回到 doDispatchServlet 的代码片段,获取 ha 以后，先调用了 preHandle 里的 interceptors，如果返回是 true，
则执行了 ha.handle，执行了 handler。

```java
...
...
        if (!mappedHandler.applyPreHandle(processedRequest, response)) {
          return;
        }

        // Actually invoke the handler.
        mv = ha.handle(processedRequest, response, mappedHandler.getHandler());

        if (asyncManager.isConcurrentHandlingStarted()) {
          return;
        }
...
...
```

# 3.1 RequestMappingHandlerAdapterd 的 handler 方法

```java
	@Override
	@Nullable
	public final ModelAndView handle(HttpServletRequest request, HttpServletResponse response, Object handler)
			throws Exception {

		return handleInternal(request, response, (HandlerMethod) handler);
	}
```

handlerInternal 调用了 invokeHandlerMethod 执行 handlerMethod

```java
	@Override
	protected ModelAndView handleInternal(HttpServletRequest request,
			HttpServletResponse response, HandlerMethod handlerMethod) throws Exception {

		ModelAndView mav;
		checkRequest(request);

		// Execute invokeHandlerMethod in synchronized block if required.
		if (this.synchronizeOnSession) {
			HttpSession session = request.getSession(false);
			if (session != null) {
				Object mutex = WebUtils.getSessionMutex(session);
				synchronized (mutex) {
					mav = invokeHandlerMethod(request, response, handlerMethod);
				}
			}
			else {
				// No HttpSession available -> no mutex necessary
				mav = invokeHandlerMethod(request, response, handlerMethod);
			}
		}
		else {
			// No synchronization on session demanded at all...
			mav = invokeHandlerMethod(request, response, handlerMethod);
		}

		if (!response.containsHeader(HEADER_CACHE_CONTROL)) {
			if (getSessionAttributesHandler(handlerMethod).hasSessionAttributes()) {
				applyCacheSeconds(response, this.cacheSecondsForSessionAttributeHandlers);
			}
			else {
				prepareResponse(response);
			}
		}

		return mav;
	}
```

invokeHandlerMethod 组装了两个实例，
ServletInvocableHandlerMethod 对象 invocableMethod 和 ModelAndViewContainer 对象 mavContainer

把一堆必要的信息传递给了 invocableMethod,比如参数解释器 this.argumentResolvers,返回值处理器 this.retrunValueHandler 等。。。

然后调用了方法 invocableMethod.invokeAndHandle(webRequest, mavContainer);

```java
	@Nullable
	protected ModelAndView invokeHandlerMethod(HttpServletRequest request,
			HttpServletResponse response, HandlerMethod handlerMethod) throws Exception {

		ServletWebRequest webRequest = new ServletWebRequest(request, response);
		try {
			WebDataBinderFactory binderFactory = getDataBinderFactory(handlerMethod);
			ModelFactory modelFactory = getModelFactory(handlerMethod, binderFactory);

			ServletInvocableHandlerMethod invocableMethod = createInvocableHandlerMethod(handlerMethod);
			if (this.argumentResolvers != null) {
				invocableMethod.setHandlerMethodArgumentResolvers(this.argumentResolvers);
			}
			if (this.returnValueHandlers != null) {
				invocableMethod.setHandlerMethodReturnValueHandlers(this.returnValueHandlers);
			}
			invocableMethod.setDataBinderFactory(binderFactory);
			invocableMethod.setParameterNameDiscoverer(this.parameterNameDiscoverer);

			ModelAndViewContainer mavContainer = new ModelAndViewContainer();
			mavContainer.addAllAttributes(RequestContextUtils.getInputFlashMap(request));
			modelFactory.initModel(webRequest, mavContainer, invocableMethod);
			mavContainer.setIgnoreDefaultModelOnRedirect(this.ignoreDefaultModelOnRedirect);

			AsyncWebRequest asyncWebRequest = WebAsyncUtils.createAsyncWebRequest(request, response);
			asyncWebRequest.setTimeout(this.asyncRequestTimeout);

			WebAsyncManager asyncManager = WebAsyncUtils.getAsyncManager(request);
			asyncManager.setTaskExecutor(this.taskExecutor);
			asyncManager.setAsyncWebRequest(asyncWebRequest);
			asyncManager.registerCallableInterceptors(this.callableInterceptors);
			asyncManager.registerDeferredResultInterceptors(this.deferredResultInterceptors);

			if (asyncManager.hasConcurrentResult()) {
				Object result = asyncManager.getConcurrentResult();
				mavContainer = (ModelAndViewContainer) asyncManager.getConcurrentResultContext()[0];
				asyncManager.clearConcurrentResult();
				if (logger.isDebugEnabled()) {
					logger.debug("Found concurrent result value [" + result + "]");
				}
				invocableMethod = invocableMethod.wrapConcurrentResult(result);
			}

			invocableMethod.invokeAndHandle(webRequest, mavContainer);
			if (asyncManager.isConcurrentHandlingStarted()) {
				return null;
			}

			return getModelAndView(mavContainer, modelFactory, webRequest);
		}
		finally {
			webRequest.requestCompleted();
    }
  }
```

invokeAndHandle 调用了 invokeAndHandle 处理请求，如果返回值为空且请求被处理则调用 mavContainer.setRequestHandled(true)后返回。
如果返回值不为空则使用 this.returnValueHandlers.handleReturnValue 处理返回值。

```java
	public void invokeAndHandle(ServletWebRequest webRequest, ModelAndViewContainer mavContainer,
			Object... providedArgs) throws Exception {

		Object returnValue = invokeForRequest(webRequest, mavContainer, providedArgs);
		setResponseStatus(webRequest);

		if (returnValue == null) {
			if (isRequestNotModified(webRequest) || getResponseStatus() != null || mavContainer.isRequestHandled()) {
				mavContainer.setRequestHandled(true);
				return;
			}
		}
		else if (StringUtils.hasText(getResponseStatusReason())) {
			mavContainer.setRequestHandled(true);
			return;
		}

		mavContainer.setRequestHandled(false);
		Assert.state(this.returnValueHandlers != null, "No return value handlers");
		try {
			this.returnValueHandlers.handleReturnValue(
					returnValue, getReturnValueType(returnValue), mavContainer, webRequest);
		}
		catch (Exception ex) {
			if (logger.isTraceEnabled()) {
				logger.trace(getReturnValueHandlingErrorMessage("Error handling return value", returnValue), ex);
			}
			throw ex;
		}
	}
```

getMethodParameters 先调用了方法 getMethodArgumentValues(),getMethodArgumentValues 里为每一个参数检查是否有支持的参数解释器 argumentResovler，如果有的话则装备一个参数实例。如果找不到 methodResolver 的话就抛出异常。
得到参数列表 args 以后调用 doInvoke()方法。

```java
  public Object invokeForRequest(NativeWebRequest request, @Nullable ModelAndViewContainer mavContainer,
			Object... providedArgs) throws Exception {

		Object[] args = getMethodArgumentValues(request, mavContainer, providedArgs);
		if (logger.isTraceEnabled()) {
			logger.trace("Invoking '" + ClassUtils.getQualifiedMethodName(getMethod(), getBeanType()) +
					"' with arguments " + Arrays.toString(args));
		}
		Object returnValue = doInvoke(args);
		if (logger.isTraceEnabled()) {
			logger.trace("Method [" + ClassUtils.getQualifiedMethodName(getMethod(), getBeanType()) +
					"] returned [" + returnValue + "]");
		}
		return returnValue;
	}


  /**
	 * Get the method argument values for the current request.
	 */
	private Object[] getMethodArgumentValues(NativeWebRequest request, @Nullable ModelAndViewContainer mavContainer,
			Object... providedArgs) throws Exception {

		MethodParameter[] parameters = getMethodParameters();
		Object[] args = new Object[parameters.length];
		for (int i = 0; i < parameters.length; i++) {
			MethodParameter parameter = parameters[i];
			parameter.initParameterNameDiscovery(this.parameterNameDiscoverer);
			args[i] = resolveProvidedArgument(parameter, providedArgs);
			if (args[i] != null) {
				continue;
			}
			if (this.argumentResolvers.supportsParameter(parameter)) {
				try {
					args[i] = this.argumentResolvers.resolveArgument(
							parameter, mavContainer, request, this.dataBinderFactory);
					continue;
				}
				catch (Exception ex) {
					if (logger.isDebugEnabled()) {
						logger.debug(getArgumentResolutionErrorMessage("Failed to resolve", i), ex);
					}
					throw ex;
				}
			}
			if (args[i] == null) {
				throw new IllegalStateException("Could not resolve method parameter at index " +
						parameter.getParameterIndex() + " in " + parameter.getExecutable().toGenericString() +
						": " + getArgumentResolutionErrorMessage("No suitable resolver for", i));
			}
		}
		return args;
	}
```

doInvoke 调用了 getBridgedMethod().invoke(getBean(), args) 执行了我们用@RequestMapping 修饰的方法，并返回了结果。

```java
public class InvocableHandlerMethod
...
...
	protected Object doInvoke(Object... args) throws Exception {
		ReflectionUtils.makeAccessible(getBridgedMethod());
		try {
			return getBridgedMethod().invoke(getBean(), args);
		}
		catch (IllegalArgumentException ex) {
			assertTargetBean(getBridgedMethod(), getBean(), args);
			String text = (ex.getMessage() != null ? ex.getMessage() : "Illegal argument");
			throw new IllegalStateException(getInvocationErrorMessage(text, args), ex);
		}
		catch (InvocationTargetException ex) {
			// Unwrap for HandlerExceptionResolvers ...
			Throwable targetException = ex.getTargetException();
			if (targetException instanceof RuntimeException) {
				throw (RuntimeException) targetException;
			}
			else if (targetException instanceof Error) {
				throw (Error) targetException;
			}
			else if (targetException instanceof Exception) {
				throw (Exception) targetException;
			}
			else {
				String text = getInvocationErrorMessage("Failed to invoke handler method", args);
				throw new IllegalStateException(text, targetException);
			}
		}
	}
  ...
  ...
}
```

# 3.2 处理返回值
