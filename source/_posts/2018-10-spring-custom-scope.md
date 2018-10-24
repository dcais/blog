---
title: Spring自定义Scope (译)
date: 2018-10-24 09:16:21
tags: spring scope java scope 译
categories: spring 
---

# 1. 概述

开箱即用的spring boot提供了"singleton"和"prototype"2个标准的，可以在任何spring application中使用的bean scope，
以及"request","session","globalSession" 3个附加的，只能在web-aware application中使用的bean scope。

标准的bean scope 不能被overridden ,web-aware application虽然可以被overiddde，可是常会带来不好的结果，所以不建议去改写。但我们也常常会遇到一些需求是预提供的bean scope满足不了的，需要额外的功能。

比如，需要开发一个multi-tenant(多租户)系统，你需要为每一个tenant提供一组隔离的bean。spring为了支持这类需求，提供了创建自定义Scope的机制。

在这篇教程中，将阐述怎样在spring中 **创建,注册,使用** 自定义bean scope。

# 2. 创建一个自定义Scope类

为了创建一个自定义类，我们需要implement Scope Interface,
并且因为会被并发调用，必须确保这个实现是进程安全(thread safe)的。

# 2.1 管理 Scope Object Callback

实现自定义Scope首先要考虑怎样存储和管理scoped object 已经 destuction callbacks。我们可以使用map或专用的类。
举个例子，本教程使用了线程安全的 synchronized maps.
让我们开始定义我们的scope类

```java
public class TenantScope implements Scope {
    private Map<String, Object> scopedObjects
      = Collections.synchronizedMap(new HashMap<String, Object>());
    private Map<String, Runnable> destructionCallbacks
      = Collections.synchronizedMap(new HashMap<String, Runnable>());
...
}
```

# 2.2 从Scope中获取Object

为了用name从Scope获取Object，我们需要实现getObject方法，**如果取不到Object，我们必须新建一个Object并返回它**

在我们的实现中，我们先检查是否能从我们的map中取到Object，如果取到了返回它，如果没取到，我们使用ObjectFactory创建一个新的Object，把它添加到map中并返回。

```java
@Override
public Object get(String name, ObjectFactory<?> objectFactory) {
    if(!scopedObjects.containsKey(name)) {
        scopedObjects.put(name, objectFactory.getObject());
    }
    return scopedObjects.get(name);
}
```

在Scope接口中定义的5个方法中，**只有get方法是必须要实现的**，其他4个方法的实现是可选的，当没有实现却被调用的情况下会抛出UnsupportedOperationException异常。

# 2.3 实现销毁回掉（Destruction Callback）

我们必须实现registerDestructionCallback方法，这个方法提供了当object或scope本身被销毁的时候的回调。

```java
@Override
public void registerDestructionCallback(String name, Runnable callback) {
    destructionCallbacks.put(name, callback);
}
```

# 2.4 从Scope移除Object

接下来，让我们实习那remove方法。remove方法从scope删除了object，移除了之前注册的销毁时的回调，并且返回被移除的object

```java
@Override
public Object remove(String name) {
    destructionCallbacks.remove(name);
    return scopedObjects.remove(name);
}
```

注意： **是调用此方法的caller去真正的执行callback并销毁被移除的object**

# 2.5 获取Conversation ID

现在，让我们实现getConversationId方法，如果你的scope支持conversation ID的概念,
你可以在这里返回，如果不支持，返回null就可以。

```java
@Override
public String getConversationId() {
    return "tenant";
}
```

# 2.6 Resolving Contextual Objects

最后，让我们实现resolveContextualObject,如果你的Scope支持多个contextual object，你需要用键值对关联每个object,并返回调用参数key所对应的object。
如果不支持，返回null就可以了。

```java
@Override
public Object resolveContextualObject(String key) {
    return null;
}
```

# 3. 注册自定义Scope

为了让spring容器意识到你的新Scope，**我们需要调用ConfigurableBeanFactory实例的register方法中注册新Scope**. 我们来看一个这个方法的定义

```java
void registerScope(String scopeName, Scope scope);
```

第一个参数scopeName是用来定义scope的唯一键，第二个参数scope是是你想要注册使用的自定义Scope的实例。

让我们创建一个自定义一个BeanFactoryPostProcessor，然后使用ConfigurableListableBeanFactory注册自定义scope

```java
public class TenantBeanFactoryPostProcessor implements BeanFactoryPostProcessor {
 
    @Override
    public void postProcessBeanFactory(ConfigurableListableBeanFactory factory) throws BeansException {
        factory.registerScope("tenant", new TenantScope());
    }
}
```

现在，让我们写一个Spring configuration类加载我们的 BeanFactoryPostProcessor实现。

```java
@Configuration
public class TenantScopeConfig {
 
    @Bean
    public static BeanFactoryPostProcessor beanFactoryPostProcessor() {
        return new TenantBeanFactoryPostProcessor();
    }
}
```

# 4. 使用自定义Scope

至此，我们已经注册了自定义scope，我们可以像使用任何scope一样使用我们的自定义scope。

先让我们定义一个TenantBean类，我们将会使用tenant-scope注入它。

```java
public class TenantBean {

    private final String name;

    public TenantBean(String name) {
        this.name = name;
    }

    public void sayHello() {
        System.out.println(
          String.format("Hello from %s of type %s",
          this.name, 
          this.getClass().getName()));
    }
}
```

注意我们在这个类上没有使用类级别的@Component和@Scope注解。现在我们在configuration类中定义tenant-scoped beans

```java
@Configuration
public class TenantBeansConfig {
    @Scope(scopeName = "tenant")
    @Bean
    public TenantBean foo() {
        return new TenantBean("foo");
    }

    @Scope(scopeName = "tenant")
    @Bean
    public TenantBean bar() {
        return new TenantBean("bar");
    }
}
```


# 5. 测试自定义Scope

让我们写一些单元测试测试一下

```java
@Test
public final void whenRegisterScopeAndBeans_thenContextContainsFooAndBar() {
    AnnotationConfigApplicationContext ctx = new AnnotationConfigApplicationContext();
    try{
        ctx.register(TenantScopeConfig.class);
        ctx.register(TenantBeansConfig.class);
        ctx.refresh();

        TenantBean foo = (TenantBean) ctx.getBean("foo", TenantBean.class);
        foo.sayHello();
        TenantBean bar = (TenantBean) ctx.getBean("bar", TenantBean.class);
        bar.sayHello();
        Map<String, TenantBean> foos = ctx.getBeansOfType(TenantBean.class);

        assertThat(foo, not(equalTo(bar)));
        assertThat(foos.size(), equalTo(2));
        assertTrue(foos.containsValue(foo));
        assertTrue(foos.containsValue(bar));
 
        BeanDefinition fooDefinition = ctx.getBeanDefinition("foo");
        BeanDefinition barDefinition = ctx.getBeanDefinition("bar");

        assertThat(fooDefinition.getScope(), equalTo("tenant"));
        assertThat(barDefinition.getScope(), equalTo("tenant"));
    }
    finally {
        ctx.close();
    }
}
```

测试输出：

```log
Hello from foo of type org.baeldung.customscope.TenantBean
Hello from bar of type org.baeldung.customscope.TenantBean
```

# 6. 总结


在本教程中我们演示了spring怎样定义，注册和使用自定义scope。你可以通过阅读
[Spring Framework Reference](https://docs.spring.io/spring/docs/current/spring-framework-reference/#beans-factory-scopes-custom)了解更多细节，你也可以通过[Spring Framework](https://github.com/spring-projects/spring-framework)源码.看一下Spring是如何实现了各种Scope.

你可以[点这里](https://github.com/eugenp/tutorials/tree/master/spring-all)获取本教程代码


**[原文链接](https://www.baeldung.com/spring-custom-scope)**: https://www.baeldung.com/spring-custom-scope