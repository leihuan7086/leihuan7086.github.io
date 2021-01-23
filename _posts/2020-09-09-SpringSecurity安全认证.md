---
title: SpringSecurity安全认证
date: 2020-09-09
category: SpringCloud
tags: SpringSecurity
description: 一个成熟的系统必须具备安全认证功能，Spring家族成员SpringSecurity为我们提供能强大的安全认证、授权等功能。本文通过源码分析身份认证（Authentication）的实现原理及过程。
---
## 概述

SpringSecurity提供2个核心功能：身份认证（Authentication）、授权（Authorization）。这篇文章结合源码来分析**身份认证（Authentication）**的过程原理。

认证过程实质即是执行一条过滤链（FilterChain），官方FilterChain默认顺序执行多个过滤器（Filter），通过过滤器Filter拦截请求执行身份认证过程。关键Filter有：LogoutFilter、BasicAuthenticationFilter、SessionManagementFilter、...

## WebSecurityConfigurerAdapter

开启SpringSecurity安全认证功能，继承抽象类WebSecurityConfigurerAdapter，并添加注解@EnableWebSecurity。默认配置即可直接使用。

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig extends WebSecurityConfigurerAdapter {

    // 默认即可
    // 一般用来配置默认认证DaoAuthenticationProvider的UserDetailsService和PasswordEncoder
    @Override
    protected void configure(AuthenticationManagerBuilder auth) throws Exception {
        super.configure(auth);
    }

    // 默认即可
    // 一般配置静态资源过滤规则，也可以在configure(WebSecurity web)配置
    @Override
    public void configure(WebSecurity web) throws Exception {
        super.configure(web);
    }
	
    // 可默认，常自定义
    // 核心配置（认证方式、请求认证过滤规则、过滤器Filter）
    @Override
    protected void configure(HttpSecurity http) throws Exception {
        super.configure(http);
    }

    // 可默认，常自定义
    // 一般用来配置ProviderManager
    @Override
    protected AuthenticationManager authenticationManager() throws Exception {
        return super.authenticationManager();
    }
}
```

## 修改默认认证用户

默认用户：**user**，密码：服务启动后随机生成，控制台打印

```sh
Using generated security password: fc291b1c-cbb9-4480-ae86-5293ac20ced3
```

修改默认认证用户，只需实现UserDetailsService接口，重写loadUserByUsername方法即可

```java
// 如需自定义UserDetailsService，实现UserDetailsService即可
// 源代码在InitializeUserDetailsBeanManagerConfigurer.InitializeUserDetailsManagerConfigurer.configure()方法中
// 修改SpringSecurity默认DaoAuthenticationProvider的UserDetailsService的两种方式：
// 1. ApplicationUserDetailsService添加@Service注解注入到Spring容器
// 2. SecurityConfig.configure(AuthenticationManagerBuilder auth)添加自定义配置
// auth.userDetailsService(new ApplicationUserDetailsService())
@Service
public class ApplicationUserDetailsService implements UserDetailsService {

    // 1，密码默认使用bcrypt方式加密
    // 2，如果密码不加密，Spring容器需要实例化DelegatingPasswordEncoder，并设置defaultPasswordEncoderForMatches为NoOpPasswordEncoder
    private PasswordEncoder passwordEncoder = PasswordEncoderFactories.createDelegatingPasswordEncoder();

    // 一般设计为从数据库读取用户
    @Override
    public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {
        if (username.contains("admin") || username.contains("root")) {
            return new User(username, passwordEncoder.encode(username + "Aa123456"), Collections.EMPTY_LIST);
        }

        throw new UserAuthenticationException("username: " + username);
    }
}
```

## 自定义认证三部曲

### AbstractAuthenticationToken

1）自定义认证Token继承抽象类AbstractAuthenticationToken

```java
@Setter
@Getter
public class CustomAuthenticationToken extends AbstractAuthenticationToken {
    
    private Object principal; //认证对象
    private String credentials; //认证密钥

    public CustomAuthenticationToken() {
        super(null);
    }  
}
```



### AbstractAuthenticationProcessingFilter

1）自定义过滤Filter继承抽象类AbstractAuthenticationProcessingFilter

```java
public class CustomAuthenticationFilter extends AbstractAuthenticationProcessingFilter {

    public CustomAuthenticationFilter(String defaultFilterProcessesUrl) {
        super(defaultFilterProcessesUrl);
    }

    // 执行过滤，默认使用父类过滤流程
    @Override
    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain) throws IOException, ServletException {
        super.doFilter(request, response, chain);
    }

    // 过滤匹配器匹配成功后，尝试认证
    @Override
    public Authentication attemptAuthentication(HttpServletRequest request, HttpServletResponse response) throws AuthenticationException, IOException, ServletException {
        // 根据request构建认证Token
        AbstractAuthenticationToken authRequest = this.obtainAuthenticationToken(request);

        // 执行认证
        return this.getAuthenticationManager().authenticate(authRequest);
    }  
}
```

2）重写WebSecurityConfigurerAdapter.configure(HttpSecurity http)，将自定义过滤Filter添加到过滤链FilterChain

```java
@Override
protected void configure(HttpSecurity http) throws Exception {
//        super.configure(http);// 自定义配置，必须删掉super.configure(http);
    http.csrf().disable(); //禁用CSRF保护
    http.httpBasic(); //启用Basic认证，FilterChain中会自动添加BasicAuthenticationFilter
    http.authorizeRequests()
            .antMatchers("/debug/**").permitAll()
            .anyRequest().authenticated();
    http.addFilterBefore(this.customAuthenticationFilter(), BasicAuthenticationFilter.class); // 添加到FilterChain中BasicAuthenticationFilter的前面
}
```

### AuthenticationProvider

1）自定义认证Provider继承抽象类AuthenticationProvider

```java
public class CustomAuthenticationProvider implements AuthenticationProvider {

    private PasswordEncoder passwordEncoder = PasswordEncoderFactories.createDelegatingPasswordEncoder();

    // supports(Class<?> authentication)返回true，即自定义Token匹配成功才会执行此认证方法
    @Override
    public Authentication authenticate(Authentication authentication) throws AuthenticationException {
        String username = authentication.getName();
        String password = (String) authentication.getCredentials();

        if ((username.contains("admin") || username.contains("root")) && "Aa123456".equals(password)) {
            CustomAuthenticationToken customAuthenticationToken = new CustomAuthenticationToken();
            User user = new User(username, password, authentication.getAuthorities());
            customAuthenticationToken.setPrincipal(user);
            customAuthenticationToken.setAuthenticated(true);
            customAuthenticationToken.setDetails(authentication.getDetails());
            return customAuthenticationToken;
        }

        throw new UserAuthenticationException("username:" + username);
    }

    @Override
    public boolean supports(Class<?> authentication) {
        return CustomAuthenticationToken.class.isAssignableFrom(authentication);
    }
}
```

2）重写WebSecurityConfigurerAdapter.authenticationManager()，将自定义认证Provider添加到认证管理器ProviderManager的认证链providers

```java
@Override
protected AuthenticationManager authenticationManager() throws Exception {
    AuthenticationManager authenticationManager = super.authenticationManager();
    if (authenticationManager instanceof ProviderManager) {
        List<AuthenticationProvider> providers = ((ProviderManager) authenticationManager).getProviders();
        // 初始化时默认官方Provider只有DaoAuthenticationProvider
        if (providers.size() <= 1) {
            providers.add(this.customAuthenticationProvider());
        }
    }
    return authenticationManager;
}
```

## 关键：认证过程解析

### 官方抽象AbstractAuthenticationProcessingFilter源码

```java
public abstract class AbstractAuthenticationProcessingFilter extends GenericFilterBean
		implements ApplicationEventPublisherAware, MessageSourceAware {

    private void doFilter(HttpServletRequest request, HttpServletResponse response, FilterChain chain)
			throws IOException, ServletException {
		if (!requiresAuthentication(request, response)) { //关键1
			chain.doFilter(request, response);
			return;
		}
		try {
			Authentication authenticationResult = attemptAuthentication(request, response); //关键2
			if (authenticationResult == null) {
				// return immediately as subclass has indicated that it hasn't completed
				return;
			}
			this.sessionStrategy.onAuthentication(authenticationResult, request, response);
			// Authentication success
			if (this.continueChainBeforeSuccessfulAuthentication) {
				chain.doFilter(request, response);
			}
			successfulAuthentication(request, response, chain, authenticationResult); //关键3
		}
		catch (InternalAuthenticationServiceException failed) {
			this.logger.error("An internal error occurred while trying to authenticate the user.", failed);
			unsuccessfulAuthentication(request, response, failed);
		}
		catch (AuthenticationException ex) {
			// Authentication failed
			unsuccessfulAuthentication(request, response, ex);
		}
	}
	
	protected boolean requiresAuthentication(HttpServletRequest request, HttpServletResponse response) {
    if (this.requiresAuthenticationRequestMatcher.matches(request)) {
        return true;
    }
    if (this.logger.isTraceEnabled()) {
        this.logger
                .trace(LogMessage.format("Did not match request to %s", this.requiresAuthenticationRequestMatcher));
    }
    return false;
}
}
```

### 官方ProviderManager 源码

```java
public class ProviderManager implements AuthenticationManager, MessageSourceAware, InitializingBean {

	@Override
	public Authentication authenticate(Authentication authentication) throws AuthenticationException {
		Class<? extends Authentication> toTest = authentication.getClass();
		AuthenticationException lastException = null;
		AuthenticationException parentException = null;
		Authentication result = null;
		Authentication parentResult = null;
		int currentPosition = 0;
		int size = this.providers.size();
		for (AuthenticationProvider provider : getProviders()) {
			if (!provider.supports(toTest)) { //关键4
				continue;
			}
			if (logger.isTraceEnabled()) {
				logger.trace(LogMessage.format("Authenticating request with %s (%d/%d)",
						provider.getClass().getSimpleName(), ++currentPosition, size));
			}
			try {
				result = provider.authenticate(authentication); //关键5
				if (result != null) {
					copyDetails(authentication, result);
					break;
				}
			}
			catch (AccountStatusException | InternalAuthenticationServiceException ex) {
				prepareException(ex, authentication);
				// SEC-546: Avoid polling additional providers if auth failure is due to
				// invalid account status
				throw ex;
			}
			catch (AuthenticationException ex) {
				lastException = ex;
			}
		}
		if (result == null && this.parent != null) {
			// Allow the parent to try.
			try {
				parentResult = this.parent.authenticate(authentication);
				result = parentResult;
			}
			catch (ProviderNotFoundException ex) {
				// ignore as we will throw below if no other exception occurred prior to
				// calling parent and the parent
				// may throw ProviderNotFound even though a provider in the child already
				// handled the request
			}
			catch (AuthenticationException ex) {
				parentException = ex;
				lastException = ex;
			}
		}
		if (result != null) {
			if (this.eraseCredentialsAfterAuthentication && (result instanceof CredentialsContainer)) {
				// Authentication is complete. Remove credentials and other secret data
				// from authentication
				((CredentialsContainer) result).eraseCredentials();
			}
			// If the parent AuthenticationManager was attempted and successful then it
			// will publish an AuthenticationSuccessEvent
			// This check prevents a duplicate AuthenticationSuccessEvent if the parent
			// AuthenticationManager already published it
			if (parentResult == null) {
				this.eventPublisher.publishAuthenticationSuccess(result);
			}

			return result;
		}

		// Parent was null, or didn't authenticate (or throw an exception).
		if (lastException == null) {
			lastException = new ProviderNotFoundException(this.messages.getMessage("ProviderManager.providerNotFound",
					new Object[] { toTest.getName() }, "No AuthenticationProvider found for {0}"));
		}
		// If the parent AuthenticationManager was attempted and failed then it will
		// publish an AbstractAuthenticationFailureEvent
		// This check prevents a duplicate AbstractAuthenticationFailureEvent if the
		// parent AuthenticationManager already published it
		if (parentException == null) {
			prepareException(lastException, authentication);
		}
		throw lastException;
	}

}
```

### 分析自定义CustomAuthenticationFilter认证过程

1）CustomAuthenticationFilter.doFilter(request, response, chain)调用父类方法super.doFilter(request, response, chain)

2）执行关键1方法requiresAuthentication(request, response)，Filter中requiresAuthenticationRequestMatcher匹配请求的url，匹配失败则调用chain.doFilter(request, response)执行FilterChain中下一个Filter的doFilter方法，匹配成功则执行attemptAuthentication(request, response)认证

3）执行关键2方法attemptAuthentication(request, response)，执行子类CustomAuthenticationFilter重写的attemptAuthentication(request, response)方法尝试认证

4）执行this.getAuthenticationManager().authenticate(authRequest)，调用官方ProviderManager.authenticate(authRequest)执行认证

5）执行关键4方法provider.supports(toTest)，即自定义CustomAuthenticationProvider.supports(toTest)方法判断Token类型是否匹配，匹配失败则跳过。匹配成功则执行provider.authenticate(authentication)执行认证

6）执行关键5方法provider.authenticate(authentication)，即自定义CustomAuthenticationProvider.authenticate(authentication)认证方法，返回认证后的对象给步骤3）中的Authentication authenticationResult = attemptAuthentication(request, response)

7）执行关键3方法successfulAuthentication(request, response, chain, authenticationResult)，将认证信息存入SpringContext。当前线程可用，FIlterChain执行完后会清空SpringContext

8）最后，认证成功后的session会通过SessionManagementFilter保存起来，之后的请求通过session进行认证即可

## 总结

1）SpringSecurity安全认证功能复杂且完整，官方默认配置即可开箱即用。

2）按需配置自定义功能，配置入口即WebSecurityConfigurerAdapter的config方法。

3）附demo源码