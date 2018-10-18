[TOC] 

# spring-security

网上都说 shiro 比 security 好，分析完 shiro 后，发现确实实现的挺轻量级的，而且扩展也比较容易，他实现的代码也很漂亮  
但是后台开发通常都会用 spring ，spring-security 自然就是官方支持的，有时候为了方便，直接使用 spring-security 了，但是配置都是网上抄的  
而 spring-security 又像一个全家桶，一个配置，加了一堆东西，有时候出了问题，都不知道怎么排查  
于是，抱着弄懂原理，心理不慌的想法，开始前线的分析源码  

首先明确一个概念，安全框架的基本功能就是认证授权，至于额外的密码hash，防御攻击之类的，都是额外的操作，security 一键就给你加了一堆东西，虽然用的爽，实际上，连使用者都可能不知道到底添加了哪些功能  

## 认证 
认证的核心接口是 `AuthenticationManager`  
```java
public interface AuthenticationManager {
	Authentication authenticate(Authentication authentication)
			throws AuthenticationException;
}
```

只有一个认证方法，凭证 `Authentication` 可以代表认证成功前的请求信息，也可以表示认证成功后的调用凭证信息  
`AuthenticationManager` 内部已经有实现了 `ProviderManager`，我们通常不会去重新实现它  
```java
public Authentication authenticate(Authentication authentication)
			throws AuthenticationException {
    Class<? extends Authentication> toTest = authentication.getClass();
    AuthenticationException lastException = null;
    Authentication result = null;
    boolean debug = logger.isDebugEnabled();

    for (AuthenticationProvider provider : getProviders()) {
        if (!provider.supports(toTest)) {
            continue;
        }

        if (debug) {
            logger.debug("Authentication attempt using "
                    + provider.getClass().getName());
        }

        try {
            result = provider.authenticate(authentication);

            if (result != null) {
                copyDetails(authentication, result);
                break;
            }
        }
        catch (AccountStatusException e) {
            prepareException(e, authentication);
            // SEC-546: Avoid polling additional providers if auth failure is due to
            // invalid account status
            throw e;
        }
        catch (InternalAuthenticationServiceException e) {
            prepareException(e, authentication);
            throw e;
        }
        catch (AuthenticationException e) {
            lastException = e;
        }
    }

    if (result == null && parent != null) {
        // Allow the parent to try.
        try {
            result = parent.authenticate(authentication);
        }
        catch (ProviderNotFoundException e) {
            // ignore as we will throw below if no other exception occurred prior to
            // calling parent and the parent
            // may throw ProviderNotFound even though a provider in the child already
            // handled the request
        }
        catch (AuthenticationException e) {
            lastException = e;
        }
    }

    if (result != null) {
        if (eraseCredentialsAfterAuthentication
                && (result instanceof CredentialsContainer)) {
            // Authentication is complete. Remove credentials and other secret data
            // from authentication
            ((CredentialsContainer) result).eraseCredentials();
        }

        eventPublisher.publishAuthenticationSuccess(result);
        return result;
    }

    // Parent was null, or didn't authenticate (or throw an exception).

    if (lastException == null) {
        lastException = new ProviderNotFoundException(messages.getMessage(
                "ProviderManager.providerNotFound",
                new Object[] { toTest.getName() },
                "No AuthenticationProvider found for {0}"));
    }

    prepareException(lastException, authentication);

    throw lastException;
}
```  
逻辑很清晰，内部使用的是 `AuthenticationProvider` 接口来认证的，而我们如果要自定义的话，通常是实现这个接口，不过内部已经有很多实现了，如果可以满足需求，直接使用内部实现就可以了  
认证成功和失败都会发布对应的事件，内部的事件都是继承 `AbstractAuthenticationFailureEvent` 的，只要捕获这个抽象的事件，就可以处理认证相关的事件了，具体的事件列表如下  
```java
AuthenticationSuccessEvent
AuthenticationFailureBadCredentialsEvent
AuthenticationFailureExpiredEvent
AuthenticationFailureProviderNotFoundEvent
AuthenticationFailureDisabledEvent
AuthenticationFailureLockedEvent
AuthenticationFailureServiceExceptionEvent
AuthenticationFailureCredentialsExpiredEvent
AuthenticationFailureProxyUntrustedEvent
```

再看看 `AuthenticationProvider` 是怎么定义的  
```java
public interface AuthenticationProvider {

	Authentication authenticate(Authentication authentication)
			throws AuthenticationException;

	boolean supports(Class<?> authentication);
}

```

两个方法，其中 `supports` 的方法参数，通常是 `Authentication` 的子类，根据方法名就知道，每个 `AuthenticationProvider` 处理特定 `Authentication` 的类型，很可能是成对出现的  
security 内部已经实现了一部分了，譬如 
1. `RememberMeAuthenticationProvider` 处理 `RememberMeAuthenticationToken`  
2. `AnonymousAuthenticationProvider` 处理 `AnonymousAuthenticationToken`
3. `RunAsImplAuthenticationProvider` 处理 `RunAsUserToken`
4. `PreAuthenticatedAuthenticationProvider` 处理 `PreAuthenticatedAuthenticationToken`
5. `DaoAuthenticationProvider` 处理 `UsernamePasswordAuthenticationToken`  
6. 。。。

这里就已经出现了后台管理系统里面常用的认证方法：用户名密码认证  
`DaoAuthenticationProvider` 类是继承 `AbstractUserDetailsAuthenticationProvider` 的  
我们关注一下 `AbstractUserDetailsAuthenticationProvider` 的 `authenticate` 方法  
```java
public Authentication authenticate(Authentication authentication)
			throws AuthenticationException {
    Assert.isInstanceOf(UsernamePasswordAuthenticationToken.class, authentication,
            messages.getMessage(
                    "AbstractUserDetailsAuthenticationProvider.onlySupports",
                    "Only UsernamePasswordAuthenticationToken is supported"));

    // Determine username
    String username = (authentication.getPrincipal() == null) ? "NONE_PROVIDED"
            : authentication.getName();

    // 这里只是一个简单的 cache
    boolean cacheWasUsed = true;
    UserDetails user = this.userCache.getUserFromCache(username);

    if (user == null) {
        cacheWasUsed = false;

        try {
            // 抽象方法，子类实现
            user = retrieveUser(username,
                    (UsernamePasswordAuthenticationToken) authentication);
        }
        catch (UsernameNotFoundException notFound) {
            logger.debug("User '" + username + "' not found");

            if (hideUserNotFoundExceptions) {
                throw new BadCredentialsException(messages.getMessage(
                        "AbstractUserDetailsAuthenticationProvider.badCredentials",
                        "Bad credentials"));
            }
            else {
                throw notFound;
            }
        }

        Assert.notNull(user,
                "retrieveUser returned null - a violation of the interface contract");
    }

    try {
        // 本类实现的前置检查 检查账号的状态，是否禁用，过期之类的
        preAuthenticationChecks.check(user);
        // 抽象方法，自定义前置检查
        additionalAuthenticationChecks(user,
                (UsernamePasswordAuthenticationToken) authentication);
    }
    catch (AuthenticationException exception) {
        // 通过缓存认证失败了，尝试从正常途径获取信息再认证一次
        if (cacheWasUsed) {
            // There was a problem, so try again after checking
            // we're using latest data (i.e. not from the cache)
            cacheWasUsed = false;
            user = retrieveUser(username,
                    (UsernamePasswordAuthenticationToken) authentication);
            preAuthenticationChecks.check(user);
            additionalAuthenticationChecks(user,
                    (UsernamePasswordAuthenticationToken) authentication);
        }
        else {
            throw exception;
        }
    }

    // 本类实现的后置检查，密码是否过期了
    postAuthenticationChecks.check(user);

    if (!cacheWasUsed) {
        this.userCache.putUserInCache(user);
    }

    Object principalToReturn = user;

    if (forcePrincipalAsString) {
        principalToReturn = user.getUsername();
    }

    return createSuccessAuthentication(principalToReturn, authentication, user);
}
``` 

接下来子类重点应该实现 `retrieveUser` 和做一下其他的校验  
看看 `DaoAuthenticationProvider` 里面的实现  
```java
protected final UserDetails retrieveUser(String username,
        UsernamePasswordAuthenticationToken authentication)
        throws AuthenticationException {
    prepareTimingAttackProtection();
    try {
        // 这里就是熟悉的 UserDetailsService 了
        UserDetails loadedUser = this.getUserDetailsService().loadUserByUsername(username);
        if (loadedUser == null) {
            throw new InternalAuthenticationServiceException(
                    "UserDetailsService returned null, which is an interface contract violation");
        }
        return loadedUser;
    }
    catch (UsernameNotFoundException ex) {
        mitigateAgainstTimingAttack(authentication);
        throw ex;
    }
    catch (InternalAuthenticationServiceException ex) {
        throw ex;
    }
    catch (Exception ex) {
        throw new InternalAuthenticationServiceException(ex.getMessage(), ex);
    }
}

protected void additionalAuthenticationChecks(UserDetails userDetails,
        UsernamePasswordAuthenticationToken authentication)
        throws AuthenticationException {
    if (authentication.getCredentials() == null) {
        logger.debug("Authentication failed: no credentials provided");

        throw new BadCredentialsException(messages.getMessage(
                "AbstractUserDetailsAuthenticationProvider.badCredentials",
                "Bad credentials"));
    }

    String presentedPassword = authentication.getCredentials().toString();

    // 这里也应该比较熟悉了 PasswordEncoder
    if (!passwordEncoder.matches(presentedPassword, userDetails.getPassword())) {
        logger.debug("Authentication failed: password does not match stored value");

        throw new BadCredentialsException(messages.getMessage(
                "AbstractUserDetailsAuthenticationProvider.badCredentials",
                "Bad credentials"));
    }
}
```

所以，如果要用用户名密码认证，只要指定 `UserDetailsService` 的实现 和 `PasswordEncoder` 的实现了  
顺便说一句，`PasswordEncoder` 有一个代理实现 `DelegatingPasswordEncoder` ，包装了所有具体实现  

认证流程核心就是这些概念，但是要注意，实际使用肯定不止这些，要使用注解，就得将这些实现跟 aop 结合，要使用拦截器，就得跟 filter 结合，这些后面再细讲  


## 授权 
授权也就是访问控制，在 security 里面是在 access 包里面  
授权的核心类是 `AbstractSecurityInterceptor` (别问我怎么找到的)，这个类的名字虽然是 interceptor ， 但是跟 spring-web 里面的拦截器没关系，只是逻辑上的拦截（跟 shiro 类型），具体的实现类，可以做 aop 拦截 或者 filter 拦截  
这个类里面涉及到授权相关的很多类 
1. `AccessDecisionManager`  授权管理
2. `AfterInvocationManager` 鉴权完成后，如果这个不为null，继续鉴权
3. `AuthenticationManager` 认证管理
4. `RunAsManager`  权限认证成功后，转换角色用的，默认没有转换功能
5. `SecurityMetadataSource` 根据调用参数，判断是否需要鉴权

这个类实际上是一个辅助类，提供了 调用前 `beforeInvocation` 调用后 `finallyInvocation` 调用结束 `afterInvocation` 
我们一个一个看，首先调用前  

```java
protected InterceptorStatusToken beforeInvocation(Object object) {
    Assert.notNull(object, "Object was null");
    final boolean debug = logger.isDebugEnabled();
    // getSecureObjectClass 是一个抽象方法，由子类指定方法参数类型
    if (!getSecureObjectClass().isAssignableFrom(object.getClass())) {
        throw new IllegalArgumentException(
                "Security invocation attempted for object "
                        + object.getClass().getName()
                        + " but AbstractSecurityInterceptor only configured to support secure objects of type: "
                        + getSecureObjectClass());
    }

    // 这里有点关键，就是使用 SecurityMetadataSource 处理调用参数，判断是否需要鉴权
    Collection<ConfigAttribute> attributes = this.obtainSecurityMetadataSource()
            .getAttributes(object);

    if (attributes == null || attributes.isEmpty()) {
        // 默认是 false
        if (rejectPublicInvocations) {
            throw new IllegalArgumentException(
                    "Secure object invocation "
                            + object
                            + " was denied as public invocations are not allowed via this interceptor. "
                            + "This indicates a configuration error because the "
                            + "rejectPublicInvocations property is set to 'true'");
        }

        if (debug) {
            logger.debug("Public object - authentication not attempted");
        }

        publishEvent(new PublicInvocationEvent(object));

        // 不需要鉴权，返回 null 了
        return null; // no further work post-invocation
    }

    if (debug) {
        logger.debug("Secure object: " + object + "; Attributes: " + attributes);
    }

    if (SecurityContextHolder.getContext().getAuthentication() == null) {
        credentialsNotFound(messages.getMessage(
                "AbstractSecurityInterceptor.authenticationNotFound",
                "An Authentication object was not found in the SecurityContext"),
                object, attributes);
    }

    // 如果还没认证，那就先做认证，不同，通常，认证应该在之前就做了，设置了 SecurityContextHolder 
    Authentication authenticated = authenticateIfRequired();

    // Attempt authorization
    try {
        // 这里判断权限
        this.accessDecisionManager.decide(authenticated, object, attributes);
    }
    catch (AccessDeniedException accessDeniedException) {
        publishEvent(new AuthorizationFailureEvent(object, attributes, authenticated,
                accessDeniedException));

        throw accessDeniedException;
    }

    if (debug) {
        logger.debug("Authorization successful");
    }

    if (publishAuthorizationSuccess) {
        publishEvent(new AuthorizedEvent(object, attributes, authenticated));
    }

    // 转换用户，默认这个是返回 null 的
    // Attempt to run as a different user
    Authentication runAs = this.runAsManager.buildRunAs(authenticated, object,
            attributes);

    if (runAs == null) {
        if (debug) {
            logger.debug("RunAsManager did not change Authentication object");
        }

        // 所以正常逻辑走这里
        // no further work post-invocation
        return new InterceptorStatusToken(SecurityContextHolder.getContext(), false,
                attributes, object);
    }
    else {
        if (debug) {
            logger.debug("Switching to RunAs Authentication: " + runAs);
        }

        // 否则，就转换角色（认证凭证换了）
        SecurityContext origCtx = SecurityContextHolder.getContext();
        SecurityContextHolder.setContext(SecurityContextHolder.createEmptyContext());
        SecurityContextHolder.getContext().setAuthentication(runAs);

        // need to revert to token.Authenticated post-invocation
        return new InterceptorStatusToken(origCtx, true, attributes, object);
    }
}
```

调用后  
```java
protected void finallyInvocation(InterceptorStatusToken token) {
    if (token != null && token.isContextHolderRefreshRequired()) {
        if (logger.isDebugEnabled()) {
            logger.debug("Reverting to original Authentication: "
                    + token.getSecurityContext().getAuthentication());
        }

        SecurityContextHolder.setContext(token.getSecurityContext());
    }
}

protected Object afterInvocation(InterceptorStatusToken token, Object returnedObject) {
    if (token == null) {
        // public object
        return returnedObject;
    }

    finallyInvocation(token); // continue to clean in this method for passivity

    if (afterInvocationManager != null) {
        // Attempt after invocation handling
        try {
            returnedObject = afterInvocationManager.decide(token.getSecurityContext()
                    .getAuthentication(), token.getSecureObject(), token
                    .getAttributes(), returnedObject);
        }
        catch (AccessDeniedException accessDeniedException) {
            AuthorizationFailureEvent event = new AuthorizationFailureEvent(
                    token.getSecureObject(), token.getAttributes(), token
                            .getSecurityContext().getAuthentication(),
                    accessDeniedException);
            publishEvent(event);

            throw accessDeniedException;
        }
    }

    return returnedObject;
}
```

这两个方法的逻辑很简单，重点在 `AfterInvocationManager` 的实现，作用跟 `AccessDecisionManager` 类似，也是用来处理权限的  


抽象类下面也就是两个方向，处理注解的 aop 实现 `MethodSecurityInterceptor` 和 处理 web 的 过滤器实现 `FilterSecurityInterceptor`   
这两个实现的不同就是调用参数不同，和判断是否需要权限验证的类不同 `MethodSecurityMetadataSource`, `FilterInvocationSecurityMetadataSource`  
这个后面再看，先看重点 `AccessDecisionManager` 的实现，权限判断，主要是这个类做的  

这个里面有一个 `AccessDecisionVoter` 列表，代表一组权限鉴定器，鉴定结果有3种，授权通过，拒绝，保留意见。  
具体判断是否授权通过由子类处理，而 `AccessDecisionManager` 的具体实现，就是鉴定策略，具体实现有   
`AffirmativeBased`  只有有一个授权通过，那就授权，如果没有通过的，只要有拒绝的，直接拒绝，否则保留意见就看怎么设置的参数了
`ConsensusBased`  通过和拒绝的投票数，看哪种结果票数多，如果同票，参数处理
`UnanimousBased`  只要有拒绝，那就直接拒绝  


再来看看具体的 aop 拦截器 `MethodSecurityInterceptor`  
```java
public Class<?> getSecureObjectClass() {
    return MethodInvocation.class;
}

public Object invoke(MethodInvocation mi) throws Throwable {
    InterceptorStatusToken token = super.beforeInvocation(mi);

    Object result;
    try {
        result = mi.proceed();
    }
    finally {
        super.finallyInvocation(token);
    }
    return super.afterInvocation(token, result);
}

public MethodSecurityMetadataSource getSecurityMetadataSource() {
    return this.securityMetadataSource;
}

public SecurityMetadataSource obtainSecurityMetadataSource() {
    return this.securityMetadataSource;
}
```
这里