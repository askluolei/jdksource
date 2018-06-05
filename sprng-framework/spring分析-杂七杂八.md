# spring 分析

1. mbdToUse.prepareMethodOverrides();
2. Object bean = resolveBeforeInstantiation(beanName, mbdToUse);
 这里是扩展流程，先给 BeanPostProcessor 处理（具体是 InstantiationAwareBeanPostProcessor 的 postProcessBeforeInstantiation 方法）
 如果创建出来bean了，就继续处理 BeanPostProcessor 接口的 postProcessAfterInitialization 然后返回了，这个创建的bean是有扩展创建的
 不走下面的正常流程
 原注释 Give BeanPostProcessors a chance to return a proxy instead of the target bean instance.
3. Object beanInstance = doCreateBean(beanName, mbdToUse, args);
 这个是正常的创建流程

