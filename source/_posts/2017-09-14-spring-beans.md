---
layout: post
title: "深入 Spring Bean 生命周期"
date: 2017-09-16
tags: spring
categories: Java
---

# 实例化
Spring 扫描包下所有类，对注解的 Bean 进行实例化

# 依赖注入 
Spring 将值和 Bean 的引用注入进 Bean 对应的属性中

# setBeanName 回调   
如果 Bean 实现了 `BeanNameAware` 接口，Spring将Bean的ID传递给 `setBeanName()`方法（实现 BeanNameAware 主要是为了通过 Bean 的引用来获得 Bean 的 id，一般业务中是很少有用到 BeanId 的）

# setBeanDactory 回调   
如果 Bean 实现了 `BeanFactoryAware` 接口，Spring 将调用 `setBeanDactory` ( BeanFactory bf )方法并把 BeanFactory 容器实例作为参数传入。（实现 BeanFactoryAware 主要目的是为了获取 Spring 容器，如 Bean 通过 Spring 容器发布事件等）

# setApplicationContext 回调
如果 Bean 实现了 `ApplicationContextAwaer` 接口，Spring 容器将调用 `setApplicationContext(ApplicationContext ctx)` 方法，把应用上下文作为参数传入。(作用与 BeanFactory 类似都是为了获取 Spring 容器，不同的是 Spring 容器在调用 `setApplicationContext()` 方法时会把它自己作为 setApplicationContext 的参数传入，而 Spring 容器在调用 `setBeanDactory()` 前需要程序员自己指定（注入）setBeanDactory 里的参数BeanFactory )

# Bean 初始化前处理
如果Bean实现了 `BeanPostProcess` 接口，Spring 将调用它们的 `postProcessBeforeInitialization`（预初始化）方法（作用是在 Bean 实例创建成功后对进行增强处理，如对 Bean 进行修改，增加某个功能）

# 容器启动完毕的回调 
如果 Bean 实现了 `InitializingBean` 接口，Spring 将调用它们的 `afterPropertiesSet()` 方法，作用与在配置文件中对 Bean 使用 init-method 声明初始化的作用一样，都是在 Bean 的全部属性设置成功后执行的初始化方法。

# Bean 初始化后回调
如果 Bean 实现了 BeanPostProcess 接口，Spring 将调用它们的 `postProcessAfterInitialization()` 方法

# 应用启动完毕 
经过以上的工作后，Bean将一直驻留在应用上下文中给应用使用，直到应用上下文被销毁

# Bean 销毁调用 
如果Bean实现了DispostbleBean接口，Spring将调用它的destory方法，作用与在配置文件中对Bean使用destory-method属性的作用一样，都是在Bean实例销毁前执行的方法。