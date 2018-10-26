---
title: 利用ApplicationContextAware制作一个获取ApplicationContext的Provider
date: 2018-10-26 10:54:37
tags: [spring]
categories: [bean]
---

# 1. 定义一个继承 ApplicationContextAware 的 bean

定义一个类ApplicationContextProvider，继承ApplicationContextAware,并用@Component注册一个bean

```java
@Component
public class ApplicationContextProvider implements ApplicationContextAware {

  private ApplicationContext applicationContext;

  @Override
  public void setApplicationContext(ApplicationContext applicationContext) throws BeansException {
    this.applicationContext = applicationContext;
  }

  public ApplicationContext getContext() {
    return applicationContext;
  }

}
```

# 2. 使用

``` java
@Service
public class UseSample {
  @Autoware
  private ApplicationContextProvider applicationContextProvider;

  public void sample(){
    ApplicationContext appctx = applicationContextProvider.getContext();
    ......
    ......

  }
}

```

# 3. 实现原理

spirng初始化的bean时，将会查看这个bean是否实现了ApplicationContextAware接口。如果是，将会调用
setApplicationContext()方法。我们在ApplicationContextProvider在实现把AppContext的地址保存到了私有变量中。


spring源码:

```java
private void invokeAwareInterfaces(Object bean) {
        .....
 if (bean instanceof ApplicationContextAware) {
  ((ApplicationContextAware)bean).setApplicationContext(this.applicationContext);
   }
}
```