---
layout: post
title: ContentProvider运行在哪个线程？
---

Android中的ContentProvider实现了对数据存储的封装，通过query、update、insert、delete等接口对外提供数据的查询和增删改等操作。假设现在有一个app实现了一个ContentProvider，
那么当其他线程或进程通过ContentResolver请求此ContentProvider进行数据操作时，ContentProvider中响应请求的代码会运行在哪个线程中呢？可以分两种情况来回答这个问题。
<!-- more -->

1. 发出请求的线程和ContentProvider处于同一个进程

   这种情况一般出现于实现ContentProvider的app中的某一个线程通过ContentResolver发出请求。此时，ContentProvider中响应请求的代码会运行在发出请求的线程中，如图1所示。
   ![图1]({{ site.url }}/lib/images/content_provider_1.png)
   *图1*

2. 发出请求的线程和ContentProvider处于不同进程

   这种情况一般出现于另一个app通过ContentResolver向实现了ContentProvider的app发出请求。此时会涉及Android的进程间通信机制Binder，而ContentProvider中响应请求的代码会运行在一个新建的名称类似`Thread_Binder`的线程中，如图2所示。
   ![图2]({{ site.url }}/lib/images/content_provider_2.png)
   *图2*

