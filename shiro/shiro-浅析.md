# shiro 分析
shiro 的核心接口就是 `SecurityManager` , 所有的实现，全是围绕这个接口的  
这个接口继承了 `Authenticator` 认证接口 , `Authorizer` 权限接口 , `SessionManager` session 管理接口 
我们先分别看看这些接口的定义  

```java
public interface Authenticator {
    public AuthenticationInfo authenticate(AuthenticationToken authenticationToken)
            throws AuthenticationException;
}

```

认证接口，是有一个方法，token 换成  info ，虽然方法名是 `authenticate` 但是，认证流程不是在这里实现的（我们后面具体分析）  
先说 token 和 info 的 区别， token 通常是用户(请求)带过来的凭证（用户名密码，accessToken 等）,默认是 `UsernamePasswordToken` 实现  
这里可以自己扩展，而 info 是什么呢？ 凭证是请求带的，不一定是真的有效的凭证，我们需要根据这个凭证去数据库（缓存）去查询系统信息  
简单点，凭证是用户名密码， info 就是根据用户名密码，返回这个用户的认证信息 用户名，加密（hash）过的密码，还有其他信息（角色，权限）  

```java
public interface Authorizer {

    boolean isPermitted(PrincipalCollection principals, String permission);

    boolean isPermitted(PrincipalCollection subjectPrincipal, Permission permission);

    boolean[] isPermitted(PrincipalCollection subjectPrincipal, String... permissions);

    boolean[] isPermitted(PrincipalCollection subjectPrincipal, List<Permission> permissions);

    boolean isPermittedAll(PrincipalCollection subjectPrincipal, String... permissions);

    boolean isPermittedAll(PrincipalCollection subjectPrincipal, Collection<Permission> permissions);

    void checkPermission(PrincipalCollection subjectPrincipal, String permission) throws AuthorizationException;

    void checkPermission(PrincipalCollection subjectPrincipal, Permission permission) throws AuthorizationException;

    void checkPermissions(PrincipalCollection subjectPrincipal, String... permissions) throws AuthorizationException;

    void checkPermissions(PrincipalCollection subjectPrincipal, Collection<Permission> permissions) throws AuthorizationException;

    boolean hasRole(PrincipalCollection subjectPrincipal, String roleIdentifier);

    boolean[] hasRoles(PrincipalCollection subjectPrincipal, List<String> roleIdentifiers);

    boolean hasAllRoles(PrincipalCollection subjectPrincipal, Collection<String> roleIdentifiers);

    void checkRole(PrincipalCollection subjectPrincipal, String roleIdentifier) throws AuthorizationException;

    void checkRoles(PrincipalCollection subjectPrincipal, Collection<String> roleIdentifiers) throws AuthorizationException;

    void checkRoles(PrincipalCollection subjectPrincipal, String... roleIdentifiers) throws AuthorizationException;
    
}
```
权限这个接口方法很多，因为权限可以根据角色和权限两个维度控制，而权限，除了可以使用字符串（应该叫权限表达式）,在内部，还以对象 `Permission` 标识  
只要记住，这个接口就是用来验证权限的  

```java
public interface SessionManager {

    Session start(SessionContext context);

    Session getSession(SessionKey key) throws SessionException;
}

```

session 管理，但是要注意的是，这里的 session 跟 servlet 里面的session 不是一个东西，虽然这里的 session 也可以支持 servlet 的  
但是，这个 session 也可以用在非web环境里面  
其中 `SessionContext` 是继承 Map 的 主要有 host 和 sessionId 两个属性（接口的 get set）  
`SessionKey` 这个里面就只有一个 `getSessionId` 方法了  

父类接口看完了，接下来看看 `SecurityManager` 里面的方法  
```java
public interface SecurityManager extends Authenticator, Authorizer, SessionManager {

    Subject login(Subject subject, AuthenticationToken authenticationToken) throws AuthenticationException;

    void logout(Subject subject);

    Subject createSubject(SubjectContext context);

}
```

方法很少，登录，登出，创建 `Subject` （接口），这个对象代表用户，他本身可以验证权限  
`SubjectContext` 里面继承 Map 添加了一些属性的 get set 方法，比如 `SecurityManager`, `sessionId`, `Subject` ... 可以自己看  
下面看看 `SecurityManager` 的实现，继承关系非常简单，一条直线，看类名就清楚每层是干什么的了  

`CachingSecurityManager`  
这层直接继承 `SecurityManager` 里面有一个 `CacheManager` 当调用 `setCacheManager` 会调用 `afterCacheManagerSet` 这个是空方法，子类实现  
很清楚，这一层只是加上了缓存管理，至于缓存怎么用，要看子类  

`RealmSecurityManager`  
`Realm` 这个是很关键的接口，可以代表数据源，无论认证还是授权，具体数据都是从 `Realm` 里面取的  
这里维护了一个 `Collection<Realm>` 集合，当完成 `setRealms` 调用后会触发 `afterRealmsSet` 方法
现在的默认实现是如果 `Realm` 实现了 `CacheManagerAware` 接口，就把上层的 `CacheManager` 设置进去  

`AuthenticatingSecurityManager`  
这一层看名字就大概知道了，处理认证的，内部一个 `Authenticator` 实现，用来处理认证方法  
默认的实现是 `ModularRealmAuthenticator`  是基于一个 `Realm` 的认证器，在 `afterRealmsSet` 方法，设置的 `Realm` 也会设置到这里面  
然后将认证方法 `authenticate` 委托给内部的 `Authenticator` 实现了  
后面具体看看 `ModularRealmAuthenticator` 的实现，大部分时候，我们不会修改这个实现  

`AuthorizingSecurityManager`  
跟上面认证一样，内部一个 `Authorizer` 实例，将授权相关的方法全部委托给内部实例处理，默认是 `ModularRealmAuthorizer`  
也是基于 `Realm` 的实现，后面具体看看 `ModularRealmAuthorizer` 内部实现  

`SessionsSecurityManager`  
同上，内部 `SessionManager` 的默认实现为 `DefaultSessionManager`，相关方法委托  
构造或者 set 了 `SessionManager` 后会触发 `afterSessionManagerSet` 方法，
目前的实现是，如果 `SessionManager` 实现接口 `CacheManagerAware` 就把上层的 `CacheManager` 设置进去  

`DefaultSecurityManager`  
上面三层分别处理了 `Authenticator, Authorizer, SessionManager` 接口  
这一层重点有3个属性 
`RememberMeManager`
`SubjectDAO`  默认实现 `DefaultSubjectDAO`
`SubjectFactory` 默认实现 `DefaultSubjectFactory`
这个里面要实现 `SecurityManager` 接口自己的3个方法 `login`, `logout`, `createSubject`  
1. login 
```java
public Subject login(Subject subject, AuthenticationToken token) throws AuthenticationException {
	AuthenticationInfo info;
	try {
		info = authenticate(token);
	} catch (AuthenticationException ae) {
		try {
			onFailedLogin(token, ae, subject);
		} catch (Exception e) {
			if (log.isInfoEnabled()) {
				log.info("onFailedLogin method threw an " +
						"exception.  Logging and propagating original AuthenticationException.", e);
			}
		}
		throw ae; //propagate
	}

	Subject loggedIn = createSubject(token, info, subject);

	onSuccessfulLogin(token, info, loggedIn);

	return loggedIn;
}
```
逻辑很简单，这里就不细讲了  
其中 `onFailedLogin` 和 `onSuccessfulLogin` 分别触发 `rememberMeFailedLogin`, `rememberMeFailedLogin` ，里面又是调用 `RememberMeManager` (如果存在) 的方法  

2. logout
```java
public void logout(Subject subject) {

	if (subject == null) {
		throw new IllegalArgumentException("Subject method argument cannot be null.");
	}

	beforeLogout(subject);

	PrincipalCollection principals = subject.getPrincipals();
	if (principals != null && !principals.isEmpty()) {
		if (log.isDebugEnabled()) {
			log.debug("Logging out subject with primary principal {}", principals.getPrimaryPrincipal());
		}
		Authenticator authc = getAuthenticator();
		if (authc instanceof LogoutAware) {
			((LogoutAware) authc).onLogout(principals);
		}
	}

	try {
		delete(subject);
	} catch (Exception e) {
		if (log.isDebugEnabled()) {
			String msg = "Unable to cleanly unbind Subject.  Ignoring (logging out).";
			log.debug(msg, e);
		}
	} finally {
		try {
			stopSession(subject);
		} catch (Exception e) {
			if (log.isDebugEnabled()) {
				String msg = "Unable to cleanly stop Session for Subject [" + subject.getPrincipal() + "] " +
						"Ignoring (logging out).";
				log.debug(msg, e);
			}
		}
	}
}
```
逻辑也不复杂，先触发 `beforeLogout` 方法，当前实现是 `rememberMeLogout` 调用 `RememberMeManager`  
然后如果认证器需要感知登出操作 `LogoutAware` 也去触发  
删除 subject 和 停止 session  
这里面可以看到 `SubjectDAO` 主要做 subject 的增删改查  

3. createSubject
```java
public Subject createSubject(SubjectContext subjectContext) {
	//create a copy so we don't modify the argument's backing map:
	SubjectContext context = copy(subjectContext);

	//ensure that the context has a SecurityManager instance, and if not, add one:
	context = ensureSecurityManager(context);
	
	// 重点在这里，这里会拿到 session （session 存储在服务端，根据 sessionId（sessionKey） 来获取）
	context = resolveSession(context);

	//Similarly, the SubjectFactory should not require any concept of RememberMe - translate that here first
	//if possible before handing off to the SubjectFactory:
	context = resolvePrincipals(context);

	// 这里创建的时候，会把 session 里面的一些信息带出来
	Subject subject = doCreateSubject(context);

	//save this subject for future reference if necessary:
	//(this is needed here in case rememberMe principals were resolved and they need to be stored in the
	//session, so we don't constantly rehydrate the rememberMe PrincipalCollection on every operation).
	//Added in 1.2:
	save(subject);

	return subject;
}
```
登录成功后，会创建 `SubjectContext` 然后调用这个方法  
后面可以详细看看 `SubjectDAO` `SubjectFactory`, `RememberMeManager`(没有默认值)  
到这里，就已经可以用了，最后一层是对 web 应用的适配  

`DefaultWebSecurityManager` 
这里面将一些相关类，全部换成 web 的实现了（大部分都是从之前的类继承下来的）  
后面详看 


## 实现细节 

### 认证 
之前说过了，认证是委托给内部实现了，我们直接看具体实现 `ModularRealmAuthenticator` 
继承 `AbstractAuthenticator` 这里面维护 `AuthenticationListener` 列表，在登录成功，失败，和登出的时候用来回调（观察者模式）, 
同时实现了 `authenticate` 固定了登录实现流程，留出一个 `doAuthenticate` 抽象方法子类实现（模版）  

`ModularRealmAuthenticator` 是基于 `Realm` 的  
```java
protected AuthenticationInfo doAuthenticate(AuthenticationToken authenticationToken) throws AuthenticationException {
	assertRealmsConfigured();
	Collection<Realm> realms = getRealms();
	if (realms.size() == 1) {
		return doSingleRealmAuthentication(realms.iterator().next(), authenticationToken);
	} else {
		return doMultiRealmAuthentication(realms, authenticationToken);
	}
}
``` 
其中 `doSingleRealmAuthentication` 好里面，只有一个认证数据源的时候  
如果 `realm` 没返回 info 直接异常 
`doMultiRealmAuthentication` 就涉及到认证策略了，其中有至少一次，第一次成功，全部成功  
策略接口是 `AuthenticationStrategy` 里面有四个方法，
1. 所有 realm 认证之前
2. 每个 realm 认证之前
3. 每个 realm 认证之后
4. 所有 realm 认证之后 
通过这4个方法，在合适的地方抛异常，就可以实现各种策略了，通常不需要我们自己重写，或者自己实现策略  
默认是至少一个认证成功  

我们看看 `Realm` 接口 
```java
public interface Realm {

    String getName();

    boolean supports(AuthenticationToken token);

    AuthenticationInfo getAuthenticationInfo(AuthenticationToken token) throws AuthenticationException;

}

```
其中就有从 token 获取 info 的接口，所以认证就很简单了  

### 权限验证  
`ModularRealmAuthorizer` 内部实现  
里面主要涉及有两个相关类 `PermissionResolver` `RolePermissionResolver` 
至于权限验证，还是 `Realm` 做的，找到 `Realm` 里面也实现 `Authorizer` 的去验证  
针对多值交易，只要任意一个 `Reaml` 里面认证通过  
```java
public boolean isPermitted(PrincipalCollection principals, Permission permission) {
	assertRealmsConfigured();
	for (Realm realm : getRealms()) {
		if (!(realm instanceof Authorizer)) continue;
		if (((Authorizer) realm).isPermitted(principals, permission)) {
			return true;
		}
	}
	return false;
}

public boolean[] isPermitted(PrincipalCollection principals, String... permissions) {
	assertRealmsConfigured();
	if (permissions != null && permissions.length > 0) {
		boolean[] isPermitted = new boolean[permissions.length];
		for (int i = 0; i < permissions.length; i++) {
			isPermitted[i] = isPermitted(principals, permissions[i]);
		}
		return isPermitted;
	}
	return new boolean[0];
}
```
瞅一眼代码就清楚了，那么现在无聊是 认证，还是授权，关键又在 `Realm` 那里去了  

### Realm
我们需要看看 `Realm` 的继承体系了  
跟 manage 的继承体系一样，
首先是 `CachingRealm` 获取 `CacheManager`
接着  `AuthenticatingRealm` 

这里有认证流程  
```java
public final AuthenticationInfo getAuthenticationInfo(AuthenticationToken token) throws AuthenticationException {
	AuthenticationInfo info = getCachedAuthenticationInfo(token);
	if (info == null) {
		//otherwise not cached, perform the lookup:
		info = doGetAuthenticationInfo(token);
		log.debug("Looked up AuthenticationInfo [{}] from doGetAuthenticationInfo", info);
		if (token != null && info != null) {
			cacheAuthenticationInfoIfPossible(token, info);
		}
	} else {
		log.debug("Using cached authentication info [{}] to perform credentials matching.", info);
	}

	if (info != null) {
		assertCredentialsMatch(token, info);
	} else {
		log.debug("No AuthenticationInfo found for submitted AuthenticationToken [{}].  Returning null.", token);
	}

	return info;
}
```
 
上层就是调用这里，先从缓存里面获取，没有就调用 `doGetAuthenticationInfo` 方法（抽象）  
有返回值，就对比 Credentials 
```java
protected void assertCredentialsMatch(AuthenticationToken token, AuthenticationInfo info) throws AuthenticationException {
	CredentialsMatcher cm = getCredentialsMatcher();
	if (cm != null) {
		if (!cm.doCredentialsMatch(token, info)) {
			//not successful - throw an exception to indicate this:
			String msg = "Submitted credentials for token [" + token + "] did not match the expected credentials.";
			throw new IncorrectCredentialsException(msg);
		}
	} else {
		throw new AuthenticationException("A CredentialsMatcher must be configured in order to verify " +
				"credentials during authentication.  If you do not wish for credentials to be examined, you " +
				"can configure an " + AllowAllCredentialsMatcher.class.getName() + " instance.");
	}
}
```	

具体怎么对比的，后面可以看看 `CredentialsMatcher` 实现，里面有一套基于密码 hash 的对比方式  
顺便说一句，这里的 cache 的 key 是 Principal 对象，自定义的对象一定要重新 hashcode 方法  

最后一层就是 
`AuthorizingRealm` 
这里就是验证权限的  
权限表达式的解析由 `PermissionResolver` 处理，里面默认实现是 `WildcardPermissionResolver`  
规则是 resource:operation  可以多个 : 分割 ,包含多个操作 * 包含所有  
验证权限，首先得获取到权限数据源里面的权限 `doGetAuthorizationInfo` 留了一个抽象方法，通过凭证(用户名)来获取权限信息  
这一层主要处理缓存和匹配   
到这里就已经结束了，这下面的实现就是具体实现了，譬如 `JdbcRealm` 等。  
通常我们都是自定义 `Realm` 只要继承这个类 `AuthorizingRealm` 重写获取认证信息和权限信息的方法就可以了  

### Subject 
这是一个很重要的接口，需要单独拿出来讲 `SecurityUtils.getSubject` 获取的当前用户，总是非 null 的  
但是非null，不代表已经认证过了，或者有权限， 如果当前用户没有登录，是会新建一个的  
通过 subject 对象 可以做登录验证权限操作，和登出操作  
其实他是包装了 `SecurityManager` 对象，只是用来表示用户而已  


其实到这里 shiro 的核心包已经看完了  
但是 shiro 可以跟 web 集成，有 shiro-web 包   
这个里面其实就一堆过滤器有点有，但是过滤器里面人认证权限校验，还是使用 `SecurityUtils.getSubject` 获取当前用户  
然后使用 `Subject` 的 api 进行权限校验，至于其他的 web 相关的类，大部分都是继承了上面的那些，多了 `Request` 和 `Response` 一般都不会用这里的对象的（不排除会用，也可以仔细看看） 

web 里面重点的有 `AbstractShiroFilter` 拦截器 ，	
```java
protected void doFilterInternal(ServletRequest servletRequest, ServletResponse servletResponse, final FilterChain chain)
            throws ServletException, IOException {

	Throwable t = null;

	try {
		final ServletRequest request = prepareServletRequest(servletRequest, servletResponse, chain);
		final ServletResponse response = prepareServletResponse(request, servletResponse, chain);

		final Subject subject = createSubject(request, response);

		//noinspection unchecked
		subject.execute(new Callable() {
			public Object call() throws Exception {
				updateSessionLastAccessTime(request, response);
				executeChain(request, response, chain);
				return null;
			}
		});
	} catch (ExecutionException ex) {
		t = ex.getCause();
	} catch (Throwable throwable) {
		t = throwable;
	}

	if (t != null) {
		if (t instanceof ServletException) {
			throw (ServletException) t;
		}
		if (t instanceof IOException) {
			throw (IOException) t;
		}
		//otherwise it's not one of the two exceptions expected by the filter method signature - wrap it in one:
		String msg = "Filtered request failed.";
		throw new ServletException(msg, t);
	}
}
```

它会在进入拦截器之前创建 subject 并且绑定当前线程，这样 `SecurityUtils.getSubject` 就可以正常使用了  
这里是根据 session 实现的，如果自定义(jjwt) , 那就需要自己定义拦截器，放在最前面进行处理


认证授权，除了可以用在 url 拦截上，还可以用在 方法上，使用 aop （shiro 注解的封装感觉很不错）  
web 相关的，可以自己细看，下面重点说说与 spring 集成  


### spring
shiro-spring 里面的内容很少  
重要的只有几个东西  
1. LifecycleBeanPostProcessor 这个是用来处理 shiro 内置的生命周期的，将这个注册为 spring 的 bean 那么 shiro 里面的生命周期，就跟 spring bean 的生命周期绑定了 
主要是 shiro 的两个接口的实现类，都需要注册到 bean ，来处理生命周期 主要是 `SecurityManager` 和 `Realm`  还有 `Environment` 的实现的实现  
值得注意的一点是，如果你不在乎生命周期，这些，你都可以不注册为 bean 

2. AuthorizationAttributeSourceAdvisor  这个是用来处理 shiro 的注解的，借助 spring-aop 实现的，至于原理，在 aop 里面已经分析过了
注解是用来保证方法级别的权限，通常不是必须的，因为功能是通过 http 接口开放出去的，如果有 rpc 相关的，可以使用这个注解
因此，只有当你确实需要使用注解，才将这个类注册为 bean， 否则，这个不是必须的 

3. `ShiroFilterFactoryBean` 这个是用来注册 shiro 的 Filter 的，用来在 http 调用接口的层面做权限控制，web 项目，通常这个是必须要配置的  
如果你只想用 注解的方式，这个也是可以不注册为 bean 的，里面的 shiro 的 filter 都有 name 的，具体可以看内部的 `DefaultFilter` 类  
```java
anon(AnonymousFilter.class),
authc(FormAuthenticationFilter.class),
authcBasic(BasicHttpAuthenticationFilter.class),
logout(LogoutFilter.class),
noSessionCreation(NoSessionCreationFilter.class),
perms(PermissionsAuthorizationFilter.class),
port(PortFilter.class),
rest(HttpMethodPermissionFilter.class),
roles(RolesAuthorizationFilter.class),
ssl(SslFilter.class),
user(UserFilter.class);
```

顺便说一句，这里是  factoryBean，他的 getObject 方法返回的实例，才是真实使用的,实际上是 `AbstractShiroFilter` 类型的  
```java
private static final class SpringShiroFilter extends AbstractShiroFilter {

	protected SpringShiroFilter(WebSecurityManager webSecurityManager, FilterChainResolver resolver) {
		super();
		if (webSecurityManager == null) {
			throw new IllegalArgumentException("WebSecurityManager property cannot be null.");
		}
		setSecurityManager(webSecurityManager);
		if (resolver != null) {
			setFilterChainResolver(resolver);
		}
	}
}
```	
