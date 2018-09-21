# spring-security

网上都说 shiro 比 security 好，分析完 shiro 后，发现确实实现的挺轻量级的，而且扩展也比较容易，他实现的代码也很漂亮  
但是后台开发通常都会用 spring ，spring-security 自然就是官方支持的，有时候为了方便，直接使用 spring-security 了，但是配置都是网上抄的  
而 spring-security 又像一个全家桶，一个配置，加了一堆东西，有时候出了问题，都不知道怎么排查  
于是，抱着弄懂原理，心理不慌的想法，开始前线的分析源码  

