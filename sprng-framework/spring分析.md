[TOC]

# spring 实现分析
分析实现细节。
spring 内部的bean 都是使用 BeanDefinition 包装了
第一步就是找到定义的地方（classpath，file，net），然后加载（annotation，xml，groovy），加载就是将BeanDefinition 添加到 spring 中，最后装配。
我们先从 BeanDefinition 开始，看看 BeanDefinition 内部结构

## BeanDefinition 内部结构
