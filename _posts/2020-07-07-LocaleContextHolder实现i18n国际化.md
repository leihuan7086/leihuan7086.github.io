---
title: LocaleContextHolder实现i18n国际化
date: 2020-07-07
category: SpringCloud
tags: i18n
description: 一个系统根据不同语言用户登录呈现出不同语言的内容，这就是i18n国际化设计。i18n国际化中最为关键的因素是语言策略Locale。如何根据不同的用户请求，获取不同的语言策略Locale？Spring项目已经为我们提供了很好的解决思路。
---

## 背景

Oscape在国外客户现场出现*“语言策略为'en-US'的用户登录校验失败时出现中文报错弹框“*。本文从此问题展开分析Spring项目中的i18n国际化解决思路。

## 国外客户中文弹窗问题分析

1）SpringMVC中通用获取i18n国际化Locale方法LocaleContextHolder.getLocale()，但是发现Oscape中直接使用存在问题。为了Oscape系统支持i18n国际化，基础模块中增加公用方法：Configurations.getLocale()

```java
// 获取当前语言环境, 优先级：用户>数据库配置>服务器
public static Locale getLocale() {
    // 1.用户语言
    Authentication existingAuth = SecurityContextHolder.getContext().getAuthentication();
    if (existingAuth != null) {
        UnifiedUserDetails user = (UnifiedUserDetails) existingAuth.getPrincipal();
        String language = user.getProfile().getLanguage();
        String[] languages = language.split("-");
        return new Locale(languages[0], languages[1]);
    }

    // 2.数据库配置
    Configuration configuration = configurationRepository.findOneByItemId(SystemConfiguration.system_default_language.itemId);
    if (configuration != null && !ObjectUtils.isEmpty(configuration.getValue())) {
        String[] languages = configuration.getValue().split("-");
        return new Locale(languages[0], languages[1]);
    }

    // 3.服务器语言
    return Locale.getDefault();
}
```

2）代码中全部使用Configurations.getLocale()获取Locale对象即可实现国际化。

但是呢，代码全文搜索发现依然还是有部分地方使用的LocaleContextHolder.getLocale()获取Locele，所以出现了国外客户现场的中文弹框问题。例如：

```java
public String getMessage(String code, Object[] args, String defaultMessage) {
    Locale locale = LocaleContextHolder.getLocale();
    return messageSource.getMessage(code, args, defaultMessage, locale);
}

```

解决方案修改为使用Configurations.getLocale()获取Locale对象。

3）是否有更好的解决方案

分析SpringMVC中通用获取i18n国际化Locale方法LocaleContextHolder.getLocale()源码，发现SpringMVC已经为我们提供了思路，需要开发人员正确的使用。上面公用方法Configurations.getLocale只是重复遭轮子而已。

## LocaleContextHolder.getLocale()源码

```java
public static Locale getLocale() {
    LocaleContext localeContext = getLocaleContext();
    if (localeContext != null) {
        Locale locale = localeContext.getLocale();
        if (locale != null) {
            return locale;
        }
    }
    return (defaultLocale != null ? defaultLocale : Locale.getDefault());
}
```

分析上面方法，可以看到return顺序为：localeContext.getLocale()>defaultLocale>Locale.getDefault()。如果localeContext.getLocale()为用户语言，defaultLocale为数据库配置语言，此方法就和Configurations.getLocale()效果完全一样了。

## defaultLocale

查看LocaleContextHolder类的属性，发现defaultLocale是一个静态成员实例Locale

```java
public abstract class LocaleContextHolder {
	private static final ThreadLocal<LocaleContext> localeContextHolder =
			new NamedThreadLocal<LocaleContext>("LocaleContext");

	// Shared default locale at the framework level
	private static Locale defaultLocale;
    
    public static void setDefaultLocale(Locale locale) {
    	LocaleContextHolder.defaultLocale = locale;
	}
}
```

所以在服务启动的时候，初始化defaultLocale为数据库配置语言。基础模块中增加初始化方法initDefaultLocale()

```java
// 初始化系统默认语言策略
@PostConstruct
public void initDefaultLocale() {
    try {
        Configuration langConfig = configurationRepository.findOneByItemId(SystemConfiguration.system_default_language.itemId);
        if (langConfig != null && !ObjectUtils.isEmpty(langConfig.getValue())) {
            String[] languages = langConfig.getValue().trim().split("-");
            if (languages.length == 2) {
                LocaleContextHolder.setDefaultLocale(new Locale(languages[0], languages[1]));
            }
        }
    } catch (Exception e) {
        logger.error("init default locale error.", e);
    }
}
```

一般情况defaultLocale配置后就不需要修改。如果需要修改，修改数据库配置，重启服务即可。

## localeContext.getLocale()

先看看localeContext是个什么东西，查看getLocaleContext()方法

```java
public static LocaleContext getLocaleContext() {
    LocaleContext localeContext = localeContextHolder.get();
    if (localeContext == null) {
        localeContext = inheritableLocaleContextHolder.get();
    }
    return localeContext;
}
```

看到localeContext是从成员变量localeContextHolder获取

```java
public abstract class LocaleContextHolder {
	private static final ThreadLocal<LocaleContext> localeContextHolder =
			new NamedThreadLocal<LocaleContext>("LocaleContext");
    
    private static final ThreadLocal<LocaleContext> inheritableLocaleContextHolder =
			new NamedInheritableThreadLocal<LocaleContext>("LocaleContext");
｝
```

惊奇的发现，localeContextHolder设计为ThreadLocal，ThreadLocal中存储的数据在线程间隔离。如果把每个线程的用户信息Locale存到成员变量localeContextHolder，通过localeContextHolder.get()获取的即是当前线程的LocaleContext信息，就不会出现登录用户信息Locale相互覆盖的问题了。

## HandlerInterceptor+AcceptHeaderLocaleResolver

Oscape中已经设计实现了这个思路，使用拦截器+请求头解析器。拦截每一个请求，将请求request中的用户信息Locale存到LocaleContextHolder类的静态成员变量localeContextHolder

自定义请求头解析器CustomAcceptHeaderLocaleResolver，setLocale必须在请求拦截过程中调用。

```java
public class CustomAcceptHeaderLocaleResolver extends AcceptHeaderLocaleResolver {
    private Locale locale;

    @Override
    public Locale resolveLocale(HttpServletRequest request) {
        return locale;
    }

    @Override
    public void setLocale(HttpServletRequest request,
                          HttpServletResponse response, Locale locale) {
        this.locale = locale;
    }
}
```

自定义拦截器LocaleChangeInterceptor

```java
public class LocaleChangeInterceptor implements HandlerInterceptor {

    // 在业务处理器处理请求之前被调用
    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
        Authentication auth = SecurityContextHolder.getContext().getAuthentication();
        if (auth != null && auth.getPrincipal() instanceof UnifiedUserDetails) {
            UnifiedUserDetails user = (UnifiedUserDetails) auth.getPrincipal();
            String language = user.getProfile().getLanguage();
            if (language != null && language.length() > 0) {
                String[] locale = language.split("-");
                if (locale.length == 2) {
                    LocaleResolver localeResolver = RequestContextUtils.getLocaleResolver(request);
                    localeResolver.setLocale(request, response, new Locale(locale[0], locale[1]));
                }
            }
        }
        return true;
    }

    // 在业务处理器处理请求执行完成后，生成视图之前执行
    @Override
    public void postHandle(HttpServletRequest request, HttpServletResponse response, Object handler, ModelAndView modelAndView) throws Exception {
		
    }

    // 在DispatcherServlet完全处理完请求后被调用，可用于清理资源等
    @Override
    public void afterCompletion(HttpServletRequest request, HttpServletResponse response, Object handler, Exception ex) throws Exception {
```

服务启动时，自定义拦截器需要添加到拦截链中，发起一个请求，拦截链中的拦截器会依次执行

```java
@Configuration
public class CustomConfigurerAdapter extends WebMvcConfigurerAdapter {

    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(new LocaleChangeInterceptor()).addPathPatterns("/**");
        super.addInterceptors(registry);
    }
}
```

## 总结

回顾LocaleContextHolder.getLocale()获取Locale顺序，localeContext.getLocale()>defaultLocale>Locale.getDefault()

1）一个请求request过来，拦截器将request中用户Locale存到ThreadLocal<LocaleContext> localeContextHolder，仅当前线程可以获取。LocaleContextHolder.getLocale()获取当前线程的用户信息Locale

2）类似轮询无请求request的线程，或者用户没有配置语言策略，获取当前线程的用户信息Locale==null，随即获取defaultLocale

3）如果初始化系统默认语言策略initDefaultLocale异常，当前线程的用户信息Locale==null，则使用服务器语言Locale.getDefault()