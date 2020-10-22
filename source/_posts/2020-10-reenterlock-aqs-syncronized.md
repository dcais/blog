---
title: ReentrantLock & Syncronized
date: 2020-10-22 14:10:56
tags: ReentryLock AQS javaCore concurrent
---

## 先来一个ReentrantLock 与 Syncronized的比较

 \ | ReentrantLock | Syncronized 
---------|----------|---------
 锁实现机制| 依赖AQS | 监视器模式（monitor）
 灵活性| 支持响应中断、超时、尝试获取锁 | 不灵活
 释放形式| 必须显式调用unlock()释放锁 | 自动释放监视器
 锁类型 | 非公平锁&公平锁 | 非公平锁 
 条件队列 | 可关联多个条件队列  | 关联一个条件队列
 可重入性 | 可重入 | 可重入

 ## 什么是AQS

 [美团的这篇文章详细介绍了AQS的机制](https://tech.meituan.com/2019/12/05/aqs-theory-and-apply.html)
 