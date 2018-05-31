# spring 分析

前面的分析，没有分析 bean 实例化的具体流程
这里具体分析一下

```java
// 开始实例化 singletons 类型的 bean 了
// Instantiate all remaining (non-lazy-init) singletons.
beanFactory.preInstantiateSingletons();
``` 

这里的 beanFactory 是 DefaultListableBeanFactory 实例

```java

``` 
