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
RequestMappingHanderMapping 的个 handler 方法。
