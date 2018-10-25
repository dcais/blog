---
title: 自定义HandlerMethodArgumentResolver，用Cookie组装一个简单的Pojo对象
date: 2018-10-24 21:14:18
tags: [spring,handlerMethodResolver,springMvc]
categories: [spring]
---

# 1. 概述

SpringMVC为我们提供了@CookieValue来注入cookie某一个key的值，但我们常需要@RequestBody一样，把Cookie组装成一个pojo对象。
这篇教程将会演示我们怎样自定义一个HandlerMethodArgumentResolver完成从Cookie组装pojo对象的需求。

# 2. 自定义HandlerMethodArgumentResolver

# 2.1 定义Annotation

我们先定义一个Annotation，叫CookieObject。

```java
@Target(ElementType.PARAMETER)
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface CookieObject {
}
```

# 2.2 实现HandlerMethodArgumentResolver接口

再定义个CookieObjectMethodArgumentResolver类，实现HandlerMethodArgumentResolver接口
HandlerMethodArgumentResolver有两个method.

```java
boolean supportsParameter(MethodParameter parameter);
```

返回true/false,表示resolver是否支持处理该参数，我们将在他的实现里，判断参数是否携带@CookieObject.

```java
Object resolveArgument(MethodParameter parameter, @Nullable ModelAndViewContainer mavContainer,
  NativeWebRequest webRequest, @Nullable WebDataBinderFactory binderFactory) throws Exception;
```

组装参数的实现逻辑。

来，让我们看一下具体的实现

```java
public class CookieObjectMethodArgumentResolver implements HandlerMethodArgumentResolver {
    private String defaultEncoding = WebUtils.DEFAULT_CHARACTER_ENCODING;

    private String decodeInternal(HttpServletRequest request, String source) {
        String enc = determineEncoding(request);
        return UriUtils.decode(source, enc);
    }

    protected String determineEncoding(HttpServletRequest request) {
        String enc = request.getCharacterEncoding();
        if (enc == null) {
            enc = this.defaultEncoding;
        }
        return enc;
    }

    @Override
    public boolean supportsParameter(MethodParameter parameter) {
        return parameter.hasParameterAnnotation(CookieObject.class);
    }

    @Override
    public Object resolveArgument(MethodParameter parameter, ModelAndViewContainer mavContainer, NativeWebRequest webRequest, WebDataBinderFactory binderFactory) throws Exception {
        HttpServletRequest servletRequest = webRequest.getNativeRequest(HttpServletRequest.class);

        Class clazz = parameter.getParameterType();

        Object obj = clazz.newInstance();
        Cookie[] cookies = servletRequest.getCookies();
        if (cookies != null) {
            BeanInfo beanInfo = Introspector.getBeanInfo(obj.getClass());
            PropertyDescriptor[] propertyDescriptors = beanInfo.getPropertyDescriptors();
            for (PropertyDescriptor property : propertyDescriptors) {
                Method setter = property.getWriteMethod();
                Class ppClazz = property.getPropertyType();
                if (setter != null) {
                    String propName = property.getName();
                    Cookie cooike = WebUtils.getCookie(servletRequest, propName);
                    if (cooike != null) {
                        String cookieValue = decodeInternal(servletRequest, cooike.getValue());
                        Object setValue = null;
                        if (Cookie.class.isAssignableFrom(ppClazz)) {
                            setValue = cookieValue;
                        } else if (cookieValue != null && binderFactory != null) {
                            WebDataBinder binder = binderFactory.createBinder(webRequest, null, propName);
                            setValue = binder.convertIfNecessary(cookieValue, ppClazz);
                        }
                        if (setValue != null) {
                            setter.invoke(obj, setValue);
                        }
                    }
                }
            }
        }
        return obj;
    }
}
```

# 2.3 加入argumentResolvers列表

最后再SpringMVC的配置中把我们定义的CookieObjectMethodArgumentResolver实例化后加入argumentResolvers列表。

```java
@Configuration
public class WebConfig extends WebMvcConfigurationSupport {
    @Override
    protected void addArgumentResolvers(List<HandlerMethodArgumentResolver> argumentResolvers) {
        CookieObjectMethodArgumentResolver cookieObjectMethodArgumentResolver = new CookieObjectMethodArgumentResolver()
        argumentResolvers.add(cookieObjectMethodArgumentResolver());
    }
}
```

# 3. 使用

# 3.1 定义一个pojo

```java
@Data
@JsonIgnoreProperties(ignoreUnknown = true)
@EqualsAndHashCode(callSuper = false)
public class MyCookieParam  {
    private String foo;
    private String bar;
}
```

# 3.2 在Controller中接收

```java
    @RequestMapping("/testCookieObject")
    @ResponseBody
    public String testCookieObject(@CookieObject MyCookieParam cookieParam) {
      return cookieParam
    }
```