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

## AnnotationAwareAspectJAutoProxyCreator
