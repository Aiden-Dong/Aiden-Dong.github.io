---
layout:     post
title:      Spring | MVC 过滤器拦截器与监听器
subtitle:   MVC(2)过滤器拦截器与监听器
date:       2018-01-12
author:     Aiden
header-img: img/post-bg-swift2.jpg
catalog: true 			
tags:								
    - Spring
---
原文:<https://blog.csdn.net/gfd54gd5f46/article/details/75022305>

### 1. 过滤器Filter :

#### 创建 ServletFilter.java 实现 Filter 方法

```
package com.example.filter;

import java.io.IOException;
import javax.servlet.Filter;
import javax.servlet.FilterChain;
import javax.servlet.FilterConfig;
import javax.servlet.ServletException;
import javax.servlet.ServletRequest;
import javax.servlet.ServletResponse;
import javax.servlet.annotation.WebFilter;

/**
 * Servlet Filter implementation class ServletFilter
 * 注解注册过滤器：实现 javax.servlet.Filter接口
 * filterName 是过滤器的名字
 * urlPatterns 是需要过滤的请求 ，这里只过滤servlet/* 下面的所有请求
 */
@WebFilter(filterName="ServletFilter",urlPatterns="/servlet/*")
public class ServletFilter implements Filter {

    /**
     * Default constructor.
     */
    public ServletFilter() {
        // TODO Auto-generated constructor stub
    }

    /**
     * @see Filter#destroy()
     */
    public void destroy() {
        System.out.println("过滤器被销毁。");
    }

    /**
     * @see Filter#doFilter(ServletRequest, ServletResponse, FilterChain)
     */
    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain) throws IOException, ServletException {
        // TODO Auto-generated method stub
        // place your code here
        System.out.println("过滤器正在执行...");
        // pass the request along the filter chain
        chain.doFilter(request, response);
    }

    /**
     * @see Filter#init(FilterConfig)
     */
    public void init(FilterConfig fConfig) throws ServletException {
        System.out.println("初始化过滤器。");
    }

}
```

#### 测试:

```
http://localhost:8080/servlet/servlet2.action
```

![](https://img-blog.csdn.net/20170712155601108?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvZ2ZkNTRnZDVmNDY=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

#### 控制台输出:

![](https://img-blog.csdn.net/20170712155608597?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvZ2ZkNTRnZDVmNDY=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

![](https://img-blog.csdn.net/20170712155615578?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvZ2ZkNTRnZDVmNDY=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

![](https://img-blog.csdn.net/20170712155626739?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvZ2ZkNTRnZDVmNDY=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)


### 2. 监听器 Listener:

#### 创建 ContextListener.java 类

```
package com.example.listener;

import javax.servlet.ServletContextEvent;
import javax.servlet.ServletContextListener;
import javax.servlet.annotation.WebListener;

/**
 * Application Lifecycle Listener implementation class ContextListener
 * @author LingDu
 */
@WebListener
public class ContextListener implements ServletContextListener {

    /**
     * Default constructor.
     */
    public ContextListener() {
        // TODO Auto-generated constructor stub
    }

    /**
     * @see ServletContextListener#contextDestroyed(ServletContextEvent)
     */
    public void contextDestroyed(ServletContextEvent arg0)  {
         System.out.println("监听器被销毁");
    }

    /**
     * @see ServletContextListener#contextInitialized(ServletContextEvent)
     */
    public void contextInitialized(ServletContextEvent arg0)  {
         System.out.println("监听器初始化。");
    }

}
```

#### 项目启动的时候 监听器是先被初始化的:

![](https://img-blog.csdn.net/20170712155651955?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvZ2ZkNTRnZDVmNDY=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)


### 拦截器:

实现拦截器可以自定义实现`HandlerInterceptor`接口，也可以通过继承`HandlerInterceptorAdapter`类，后者是前者的实现类。
下面是拦截器的一个实现的例子，目的是判断用户是否登录。如果`preHandle`方法`return true`，则继续后续处理。

#### 创建拦截器:

```
public class LoginInterceptor extends HandlerInterceptorAdapter {
    /**
     *预处理回调方法，实现处理器的预处理（如登录检查）。
     *第三个参数为响应的处理器，即controller。
     *返回true，表示继续流程，调用下一个拦截器或者处理器。
     *返回false，表示流程中断，通过response产生响应。
     */
    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response,
            Object handler) throws Exception {
        System.out.println("-------------------preHandle");
         // 验证用户是否登陆
        Object obj = request.getSession().getAttribute("username");
        if (obj == null || !(obj instanceof String)) {
            response.sendRedirect(request.getContextPath() + "/index.html");
            return false;
        }
        return true;
    }

    /**
     *当前请求进行处理之后，也就是Controller 方法调用之后执行，
     *但是它会在DispatcherServlet 进行视图返回渲染之前被调用。
     *此时我们可以通过modelAndView对模型数据进行处理或对视图进行处理。
     */
    @Override
    public void postHandle(HttpServletRequest request, HttpServletResponse response,
            Object handler, ModelAndView modelAndView) throws Exception {
        System.out.println("-------------------postHandle");
    }

    /**
     *方法将在整个请求结束之后，也就是在DispatcherServlet 渲染了对应的视图之后执行。
     *这个方法的主要作用是用于进行资源清理工作的。
     */
    @Override
    public void afterCompletion(HttpServletRequest request, HttpServletResponse response,
            Object handler, Exception ex) throws Exception {
        System.out.println("-------------------afterCompletion");
    }

}
```

#### 2. 注册拦截器

为了使自定义的拦截器生效，需要注册拦截器到`spring`容器中，具体的做法是继承·`WebMvcConfigurerAdapter`类，覆盖其`addInterceptors(InterceptorRegistry registry)`方法。
最后别忘了把Bean注册到Spring容器中，可以选择`@Component` 或者 `@Configuration`。


```
@Component
public class InterceptorConfiguration extends WebMvcConfigurerAdapter{
    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        // 注册拦截器
        InterceptorRegistration ir = registry.addInterceptor(new LoginInterceptor());
        // 配置拦截的路径
        ir.addPathPatterns("/**");
        // 配置不拦截的路径
        ir.excludePathPatterns("/**.html");

        // 还可以在这里注册其它的拦截器
        //registry.addInterceptor(new OtherInterceptor()).addPathPatterns("/**");
    }
}
```
