---
title: 使用Spring AOP & 自定义注解 记录日志
tags: spring
categories: Framework
date: 2018-09-24 12:04:16
---


## 0x00 一切为了少写代码

最近在重构一块儿业务代码的时候发现：有好几个类中有大量重复的代码，其实都在干同一件事，就是记录方法执行所耗时间，代码大概长这个样子：

```java
public void doSomething() {
    long start = System.currentTimeMillis();
    try {
        // doSomeTimeConsumingTasks
    } finally {
        long cost = System.currentTimeMillis() - start;
        if (cost > 1000) {
            logger.info("doSomething cost {}s", cost / 1000.0);
        }
    }
}
```

数了一数，在一个方法里和具体业务并无关系的日志记录代码就占了8行。不对，在我们部门的code style里，左边的大括号是要换行的，代码应该长这个样子：

```java
public void doSomething() {
    long start = System.currentTimeMillis();
    try
    {
        // doSomeTimeConsumingTasks
    }
    finally
    {
        long cost = System.currentTimeMillis() - start;
        if (cost > 1000)
        {
            logger.info("doSomething cost {}s", cost / 1000.0);
        }
    }
}
```

现在和具体业务并无关系的日志记录代码就变成12行了，如果多写几遍这样的方法，大概一个月的代码量KPI就够了。

不过，要做一个还算有点追求的程序猿，还是要力求在实现相同功能的前提下少写代码的，毕竟代码多了会让人没有读下去的欲望，不利于后期维护。

<!-- more -->

## 0x01 高举 Spring AOP 大旗

使用过Spring AOP 的同学想必遇到这种重复日志的问题，早就计上心来：“给我一个切面，我能把你都记下来”。

最初的我也是这么想的，花了几分钟，定义了一个切面：

```java
package com.tomoyadeng.springbootutils.aop;

import org.aspectj.lang.ProceedingJoinPoint;
import org.aspectj.lang.annotation.Around;
import org.aspectj.lang.annotation.Aspect;
import org.aspectj.lang.annotation.Pointcut;
import org.aspectj.lang.reflect.MethodSignature;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.stereotype.Component;

import java.lang.reflect.Method;

@Aspect
@Component
public class LogAspect {

    @Pointcut("execution(* com.tomoyadeng.springbootutils.service.*.*(..))")
    public void logPointCut() {}

    @Around("logPointCut()")
    public Object logTimeCost(ProceedingJoinPoint pjp) throws Exception {
        MethodSignature signature = (MethodSignature) pjp.getSignature();
        // 根据方法签名拿到方法的类，并通过类获取对应的logger
        Logger logger = LoggerFactory.getLogger(signature.getMethod().getDeclaringClass());

        // 拿到函数上注解内的msg 和 threshold
        Method m = signature.getMethod();
        long start = System.currentTimeMillis();
        try {
            return pjp.proceed();
        } catch (Throwable throwable) {
            throw new Exception(throwable);
        } finally {
            long cost = System.currentTimeMillis() - start;
            // 记录日志
            if (cost > 1000) {
                logger.info("{} cost {}s", m.getName(), cost / 1000.0);
            }
        }
    }
}
```

写个测试：

```java
package com.tomoyadeng.springbootutils;

import com.tomoyadeng.springbootutils.service.CustomizeService;
import org.junit.Test;
import org.junit.runner.RunWith;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.context.junit4.SpringRunner;

@RunWith(SpringRunner.class)
@SpringBootTest
public class LogTimeCostTest {
    @Autowired private CustomizeService service;

    @Test
    public void testLogForCostTime() {

        service.someMethod();
    }
}
```

跑一下看看，果然能在日志中看到对应的日志

```console
2018-09-24 00:58:24.963  INFO 6808 --- [           main] c.t.s.service.CustomizeService           : someMethod cost 1.006s
```

日志记录算是基本完成了，不过仔细一想，这样会存在两个问题：

1. 该包下并不是所有的类的公有方法都要记录耗时日志，有没有精确控制的办法呢？
2. 不是所有的方法记录耗时日志的阈值都是1000ms，有没有参数控制的办法呢？

## 0x02 插上注解的翅膀

回想我们在开发Spring应用时，经常会使用各种各样的注解来增强功能，比如`@GetMapping`， `@ComponentScan`等，那么上面提到的问题就可以通过注解来解决了。

首先，新建一个注解

```java
package com.tomoyadeng.springbootutils.annotation;

import java.lang.annotation.*;

@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface LogTimeCost {
    // 自定义日志中写入的消息
    String msg() default "";

    // 自定义记录耗时日志的阈值
    int threshold() default 0;
}
```

随后，对之前的切面进行改造：

```java
package com.tomoyadeng.springbootutils.aop;

import com.tomoyadeng.springbootutils.annotation.LogTimeCost;
import org.aspectj.lang.ProceedingJoinPoint;
import org.aspectj.lang.annotation.Around;
import org.aspectj.lang.annotation.Aspect;
import org.aspectj.lang.annotation.Pointcut;
import org.aspectj.lang.reflect.MethodSignature;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.stereotype.Component;

import java.lang.reflect.Method;

@Aspect
@Component
public class LogAspect {

    @Pointcut("@annotation(com.tomoyadeng.springbootutils.annotation.LogTimeCost)")
    public void logPointCut() {}

    @Around("logPointCut()")
    public Object logTimeCost(ProceedingJoinPoint pjp) throws Exception {
        MethodSignature signature = (MethodSignature) pjp.getSignature();
        // 根据方法签名拿到方法的类，并通过类获取对应的logger
        Logger logger = LoggerFactory.getLogger(signature.getMethod().getDeclaringClass());

        // 拿到函数上注解内的msg 和 threshold
        Method m = signature.getMethod();
        String msg = "";
        int threshold = 0;
        if (m.isAnnotationPresent(LogTimeCost.class)) {
            LogTimeCost annotation = m.getAnnotation(LogTimeCost.class);
            msg = annotation.msg();
            threshold = annotation.threshold();
        }

        long start = System.currentTimeMillis();
        try {
            return pjp.proceed();
        } catch (Throwable throwable) {
            throw new Exception(throwable);
        } finally {
            long cost = System.currentTimeMillis() - start;
            // 记录日志
            if (cost > threshold) {
                logger.info("{} [{}] cost {}s", m.getName(), msg, cost / 1000.0);
            }
        }
    }
}
```

接下来，在需要记录耗时的方法前加上LogTimeCost注解即可

```java
@LogTimeCost(msg = "sleep 1 s", threshold = 500)
public void someMethod() {
    try {
        TimeUnit.SECONDS.sleep(1);
    } catch (InterruptedException e) {
        e.printStackTrace();
    }
}
```

最后，再跑一下测试

```console
2018-09-24 01:20:55.267  INFO 11748 --- [           main] c.t.s.service.CustomizeService           : someMethod [sleep 1 s] cost 1.007s
```
