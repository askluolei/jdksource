# spring 分析

## 简介

1. Advice 通知
 定义在连接点做什么，BeforeAdvice，AfterAdvice，ThrowsAdvice 等
2. Pointcut 切点
 定义 Advice 作用哪个连接点
3. Advisor 通知器
 组合 Advice 和 Pointcut

**动态代理**
1. JDK 的动态代理（基于接口的）
2. Cglib 的字节码增强（spring5 还有 Objenesis ）

简单点说， Advice 就是要做什么 ， Pointcut 是在哪里做，通常我们不需要关注 Advisor ，这个是 spring 内部用来连接 Advice 和 Pointcut 的
Advice 有很两种，一种是 beforeAdvice，afterAdvice，aroundAdvie 这是一类，还有 Interceptor 也是一类
通常，我们用户使用注解方式的，使用的是第一类，spring 内部扩展，使用的是第二类，譬如 spring-tx 事务


