---
layout:     post
title:      "傻傻分不清的过滤器和拦截器"
subtitle:   "filter、interceptor"
date:       2021-01-21
author:     "ThreeJin"
header-mask: 0.5
catalog: true
header-img: "https://gitee.com/liaoxinyiqiqi/my-blog-images/raw/master/img/websocket02.jpg"
tags:
    - Spring
    - Java
---
> 资料来源于网络上各位前辈（草木物语、KINBUG-BP等）

### 前言
之前在获取Websocket的请求ip的时候，用的方法是filter过滤Websocket的连接请求，然后从请求中进行预处理获取。所以就想到了很久之前复习过的filter和interceptor，然后又联想到interceptor和AOP，所以这里进行归纳总结一下。
### Filter
Filter 是 JavaEE 中 Servlet 规范的一个组件，位于包`javax.servlet` 中，它可以在 HTTP 请求到达 Servlet 之前，被一个或多个Filter处理。Filter的这个特性在生产环境中有很广泛的应用，如：修改请求和响应、防止xss攻击、**包装二进制流使其可以多次读**（这一点在之前的项目中使用操作日志的时候有使用到，但是要考虑文件上传时候的问题），等等。

##### doFilter(ServletRequest request, ServletResponse response, FilterChain chain)
Filter过滤器实现的是javax.servlet.Filter接口的类，而在javax.servlet.Filter中定义了以下三个方法：  

![](https://gitee.com/liaoxinyiqiqi/my-blog-images/raw/master/img/java-filter-interceptor-01.jpg)  

- 需要放行该请求  
当完成了请求的预处理操作后需要放行该请求时，直接调用`filterChain.doFilter(servletRequest,servletResponse);`即可  
- 不能放过该请求  
直接在`doFilter()`方法中`return`掉即可  
- 返回之前  
往往这个操作用的时候比较少，也就是说最开始放行了请求，然后走正常的处理逻辑，在请求返回给调用方之前再进行一些预处理，此时的逻辑只需要跟在`filterChain.doFilter(servletRequest,servletResponse);`后即可   

##### destroy()
Filter对象创建后会驻留在内存，当web应用移除或服务器停止时调用destroy()方法进行销毁。在Web容器卸载 Filter 对象之前被调用。destroy()方法在Filter的生命周期中仅执行一次。通过destroy()方法，可以释放过滤器占用的资源  
##### init()
Filter的创建和销毁由WEB服务器负责。 web 应用程序启动时，web 服务器将创建Filter 的实例对象，并调用其init方法，完成对象的初始化功能，从而为后续的用户请求作好拦截的准备工作，filter对象只会创建一次，init方法也只会执行一次。通过init方法的参数，可获得代表当前filter配置信息的FilterConfig对象。  
##### 实现方式一：@WebFilter
@WebFilter 用于将一个类声明为过滤器，该注解将会在部署时被容器处理，容器将根据具体的属性配置将相应的类部署为过滤器。该注解具有下表给出的一些常用属性 ( 以下所有属性均为可选属性，但是 value、urlPatterns、servletNames 三者必需至少包含一个，且 value 和 urlPatterns 不能共存，如果同时指定，通常忽略 value 的取值 )  

![](https://gitee.com/liaoxinyiqiqi/my-blog-images/raw/master/img/java-filter-interceptor-02.jpg)  

代码示意：
```java
package com.xc.common.filter;

import java.io.IOException;

import javax.servlet.Filter;
import javax.servlet.FilterChain;
import javax.servlet.FilterConfig;
import javax.servlet.ServletException;
import javax.servlet.ServletRequest;
import javax.servlet.ServletResponse;
import javax.servlet.annotation.WebFilter;

/**
 * @ClassName: FilterDemo01
 * @Description:filter的三种典型应用： <br/>
 * 1、可以在filter中根据条件决定是否调用chain.doFilter(request, response)方法， 即是否让目标资源执行<br/>
 * 2、在让目标资源执行之前，可以对request\response作预处理，再让目标资源执行 <br/>
 * 3、在目标资源执行之后，可以捕获目标资源的执行结果，从而实现一些特殊的功能 <br/>
 */
@WebFilter(filterName = "FilterDemo01", urlPatterns = { "/*" })
public class FilterDemo01 implements Filter {

    /**
     * 不过滤的资源
     */
    private String[] nofilter;

    @Override
    public void init(FilterConfig filterConfig) throws ServletException {
        //将不过滤的资源存入数组中
        String nofilterString = filterConfig.getInitParameter("nofilter");
        if (nofilterString!=null&&nofilterString.length()>0){
            nofilter = nofilterString.split(",");
        }
    }

    @Override
    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain) throws IOException, ServletException {

        HttpServletRequest request = (HttpServletRequest) servletRequest;
        boolean flag = isNofilter(request);
        if (flag){
            // 对request和response进行一些预处理
            request.setCharacterEncoding("UTF-8");
            response.setCharacterEncoding("UTF-8");
            response.setContentType("text/html;charset=UTF-8");
    
            System.out.println("FilterDemo01执行前！！！");
            chain.doFilter(request, response); // 让目标资源执行，放行
            System.out.println("FilterDemo01执行后！！！");
        }else {
            request.getRequestDispatcher("/forword").forward(servletRequest,servletResponse);
        }

    }

    @Override
    public void destroy() {
        System.out.println("----过滤器销毁----");
    }
    
    public boolean isNofilter(HttpServletRequest request){
        String requestURI = request.getRequestURI();
        System.out.println(requestURI);
        for (String s : nofilter) {
            if (requestURI.contains(s)){
                return true;
            }
        }
        return false;
    }
}

```

**多个filter的话，通过`@Order(x)`注解来区分优先级别，x越小优先级越高**  
**启动类上加上@ServletComponentScan**  
在SpringBootApplication(启动类)上使用`@ServletComponentScan`注解后，Servlet、Filter(过滤器)、Listener(监听器)可以直接通过`@WebServlet`、`@WebFilter`、`@WebListener`注解自动注册，无需其他代码  

##### 实现方式二：实现Filter接口，然后通过FilterRegistrationBean进行注册

```java
@Configuration
public class WebConfig {

    @Bean
    public FilterRegistrationBean filterRegistrationBean(){
        FilterRegistrationBean registrationBean = new FilterRegistrationBean(new FirstFilter());
        //过滤所有路径
        registrationBean.addUrlPatterns("/*");
        //添加不过滤路径
        registrationBean.addInitParameter("noFilter","/one,/two");
        registrationBean.setName("firstFilter");
        registrationBean.setOrder(1);
        return registrationBean;
    }
}
```

`@Configuration` 用于定义配置类，可替换XML配置文件，被注解的**类内部包含一个或多个`@Bean`注解方法**。可以被`AnnotationConfigApplicationContext`或者`AnnotationConfigWebApplicationContext` 进行扫描。用于构建bean定义以及初始化Spring容器

### Interceptor
在SpringMVC中提供了类似于Filter的功能————拦截器，主要用于拦截用户的请求并做相应的处理，通常应用在权限验证、记录请求信息的日志、判断用户是否登录等功能上  
在 Spring MVC 框架中定义一个拦截器需要对拦截器进行定义和配置，定义一个拦截器可以通过两种方式：  
一种是通过实现 HandlerInterceptor 接口或继承 HandlerInterceptor 接口的实现类来定义；  
另一种是通过实现 WebRequestInterceptor 接口或继承 WebRequestInterceptor 接口的实现类来定义

类比AOP：**相对于拦截器而言，AOP更加细致，而且非常灵活，拦截器只能针对URL做拦截，而AOP针对具体的代码，能够实现更加复杂的业务逻辑**

##### preHandle()
该方法在控制器的处理请求方法前执行，其返回值表示是否中断后续操作，返回 true 表示继续向下执行，返回 false 表示中断后续操作
##### postHandle()
该方法在控制器的处理请求方法调用之后、解析视图之前执行，可以通过此方法对请求域中的模型和视图做进一步的修改
##### afterCompletion()
视图渲染结束后执行，可以通过此方法实现一些资源清理、记录日志信息等工作

##### 创建拦截器：基于URL实现

```java
@Order(1001)
public class LoginInterceptor extends HandlerInterceptorAdapter{

    public static final String NO_INTERCEPTOR_PATH =".*/((.css)|(.js)|(images)|(login)|(anon)).*";
    
	/**
     * 在请求处理之前进行调用（Controller方法调用之前）
     * 基于URL实现的拦截器
     * @param request
     * @param response
     * @param handler
     * @return
     * @throws Exception
     */
    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
        String path = request.getServletPath();
        if (path.matches(NO_INTERCEPTOR_PATH)) {
            //基于正则匹配的url
        	//不需要的拦截直接过
            return true;
        } else {
        	// 这写你拦截需要干的事儿，比如取缓存，SESSION，权限判断等
            //do something,符合要求的return true，否则return false
            return true;
        }
    }
}
```

##### 创建拦截器：基于注解

```java
//创建注解
@Target({ElementType.METHOD})// 可用在方法名上
@Retention(RetentionPolicy.RUNTIME)// 运行时有效
public @interface LoginRequired {
}

//创建拦截器
@Order(1001)
public class AuthorityInterceptor extends HandlerInterceptorAdapter{
	
	 @Override
	 public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
	 	// 如果不是映射到方法直接通过
        if (!(handler instanceof HandlerMethod)) {
            return true;
        }
        // ①:START 方法注解级拦截器
        HandlerMethod handlerMethod = (HandlerMethod) handler;
        Method method = handlerMethod.getMethod();
        // 判断接口是否需要登录
        LoginRequired methodAnnotation = method.getAnnotation(LoginRequired.class);
        // 有 @LoginRequired 注解，需要认证
        if (null!=methodAnnotation) {
            // 这写你拦截需要干的事儿，比如取缓存，SESSION，权限判断等
            //do something
            return true;
        }
        return true;
	}
}
```

##### 注册拦截器
```java
@Configuration
public class WebConfigurer implements WebMvcConfigurer {
	 @Override
	 public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(LoginInterceptor()).addPathPatterns("/**");
        registry.addInterceptor(AuthorityInterceptor()).addPathPatterns("/**");
	 }
	 
	 @Bean
	 public LoginInterceptor LoginInterceptor() {
		 return new LoginInterceptor();
	 }
	 
	 @Bean
	 public AuthorityInterceptor AuthorityInterceptor() {
		 return new AuthorityInterceptor();
	 }
}
```

### 总结
一般情况下数据被过滤的时机越早对服务的性能影响越小，因此在编写相对比较公用的代码时，优先考虑过滤器，然后是拦截器，最后是aop。  
比如权限校验，一般情况下，所有的请求都需要做登陆校验，此时就应该使用过滤器在最顶层做校验；  
日志记录，一般日志只会针对部分逻辑做日志记录，而且牵扯到业务逻辑完成前后的日志记录，因此使用过滤器不能细致地划分模块，此时应该考虑拦截器，然而拦截器也是依据URL做规则匹配，因此相对来说不够细致，因此我们可以考虑到使用AOP实现，AOP可以针对代码的方法级别做拦截，很适合日志功能。