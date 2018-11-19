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

限于偏于，这里将重点分析我们常用的 RequestMappingHandlerMapping。

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

## 1.2 RequestMappingHandlerMapping

### 1.2.1 RequestMappingHandlerMapping 类图

### 1.2.3 RequestMappingHandlerMapping 初始化

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
}
```
