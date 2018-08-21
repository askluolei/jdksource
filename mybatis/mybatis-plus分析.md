[TOC]   

# mybatis-plus 分析 
mybatis-plus 是一个对 mybatis 的增强包  
[mybatis-plus 参考文档](http://mp.baomidou.com/)    

## 笔记 
1. MybatisSessionFactoryBuilder
这个启动类，主要替换了解析 MybatisXMLConfigBuilder, 和根据原始 Configuration 添加 GlobalConfiguration

2. MybatisXMLConfigBuilder  
这个类，是为了使用自己的 MybatisConfiguration

3. MybatisConfiguration 
为了使用自定义 mapper 注册器 MybatisMapperRegistry 和 修改默认 LanguageDriver 的实现 MybatisXMLLanguageDriver

4. MybatisXMLLanguageDriver 
这个就是继承 mybatis 的默认实现 XMLLanguageDriver， 为了替换默认的 ParameterHandler 实现，换成 MybatisDefaultParameterHandler   

5. MybatisMapperRegistry    
还是继承默认的实现。为了注入 SqlRunner （直接sql执行。），替换了默认的 mapper 接口处理类为 MybatisMapperAnnotationBuilder   

6. MybatisDefaultParameterHandler   
继承默认的实现。填充插入方法主键 ID，还有 MetaObjectHandler 的插入和更新之前的自定义填充

7. MybatisMapperAnnotationBuilder   
继承 MapperAnnotationBuilder，这里主要是为了注入通用的增删改查 AutoSqlInjector  

8. 