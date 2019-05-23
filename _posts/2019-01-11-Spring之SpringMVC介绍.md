---
layout:     post
title:      Spring | MVC 技术
subtitle:   MVC(1)介绍
date:       2018-01-11
author:     Aiden
header-img: img/post-bg-swift2.jpg
catalog: true 			
tags:								
    - Spring
---

原文: [Spring MVC 框架入门学习 - 简书](https://www.jianshu.com/p/0b157b3e110b)

### 1.Spring web mvc 介绍:

Spring web mvc和Struts2都属于表现层的框架,它是Spring框架的一部分,我们可以从Spring的整体结构中看得出来:

![](https://upload-images.jianshu.io/upload_images/763193-6dbbea6d7ab9617e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/700)

### 2.Web mvc

1. 用户发出的请求通过 **HadllerMapping** 定位到指定的**控制器(Controller)**
2. **控制器(Controller)** 通过 **模型(Model)** 处理数据并得到处理结果,模型通常是指业务逻辑。
3. **模型(Model)** 在通过**视图适配器(ViewResolver)**,将Model转成用户需要的**视图(View)**。
4. 控制器将视图response响应给用户,通过视图展示给用户要的数据或处理结果。

![image.png](https://upload-images.jianshu.io/upload_images/10402860-38af0f2301984a4f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


### 3. Spring Web MVC 框架

架构图:

![image.png](https://upload-images.jianshu.io/upload_images/10402860-03c7fc280c9090d5.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/600)

**流程 :**

1. 用户发送请求至**前端控制器** `DispatcherServlet`.
2. `DispatcherServlet` 收到请求调用`HandlerMapping`**处理器映射器**.
3. 处理器映射器找到具体的处理器，生成**处理器对象**及**处理器拦截器**(如果有则生成)一并返回给`DispatcherServlet`.
4. `DispatcherServlet`调用`HandlerAdapter`**处理器适配器**.
5. `HandlerAdapter`经过适配调用具体的**处理器(Controller，也叫后端控制器)**.
6. `Controller`执行完成返回`ModelAndView`.
7. `HandlerAdapter`将`controller`执行结果`ModelAndView`返回给DispatcherServlet.
8. `DispatcherServlet`将`ModelAndView`传给`ViewReslover`**视图解析器**.
9. `ViewReslover`解析后返回具体`View`.
10. `DispatcherServlet`根据View进行渲染视图（即将模型数据填充至视图中）。
11. `DispatcherServlet`响应用户。

**组件说明:**

> 以下组件通常使用框架提供实现:

`DispatcherServlet`：作为前端控制器，整个流程控制的中心，控制其它组件执行，统一调度，降低组件之间的耦合性，提高每个组件的扩展性。
`HandlerMapping`：通过扩展处理器映射器实现不同的映射方式，例如：配置文件方式，实现接口方式，注解方式等。
`HandlAdapter`：通过扩展处理器适配器，支持更多类型的处理器。
`ViewResolver`：通过扩展视图解析器，支持更多类型的视图解析，例如：jsp、freemarker、pdf、excel等。

> 下边两个组件通常情况下需要开发:

`Handler`：处理器，即后端控制器用controller表示。
`View`：视图，即展示给用户的界面，视图中通常需要标签语言展示模型数据。

### 4.开发环境准备:

本教程使用Eclipse+tomcat7开发
详细参考“Eclipse开发环境配置-indigo.docx”文档

### 5. 第一个Spring mvc 工程

#### 5.1. 建立一个web 项目:

在eclipse下创建动态web工程springmvc_01。

#### 5.2 导入spring 包

```
<!-- spring -->
<dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-beans</artifactId>
    <version>${spring.version}</version>
</dependency>

<dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-context</artifactId>
    <version>${spring.version}</version>
</dependency>

<dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-orm</artifactId>
    <version>${spring.version}</version>
</dependency>

<dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-webmvc</artifactId>
    <version>${spring.version}</version>
</dependency>

<dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-web</artifactId>
    <version>${spring.version}</version>
</dependency>

<dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-jms</artifactId>
    <version>${spring.version}</version>
</dependency>

<dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-jdbc</artifactId>
    <version>${spring.version}</version>
</dependency>
```

#### 5.3 前端控制器(DispatcherServlet)配置

在`WEB-INF\web.xml`中配置前端控制器:

```
<servlet>
    <!-- 前端控制器的名字 -->
    <servlet-name>springmvc</servlet-name>
    <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>

    <!-- Spring mvc 配置-->
    <init-param>
        <param-name>contextConfigLocation</param-name>
        <param-value>classpath:springmvc-servlet.xml</param-value>
    </init-param>
    // 表示servlet随服务启动
    <load-on-startup>1</load-on-startup>
</servlet>

<servlet-mapping>
    <servlet-name>springmvc</servlet-name>
    // 以.action结尾的请求交给DispatcherServlet处理
    <url-pattern>*.action</url-pattern>
</servlet-mapping>
```

`load-on-startup`：表示servlet随服务启动;

`url-pattern`：`*.action`的请交给`DispatcherServlet`处理。

`contextConfigLocation`：指定`springmvc`配置的加载位置，如果不指定则默认加载`WEB-INF/[DispatcherServlet 的Servlet 名字]-servlet.xml`。


#### 5.4 SpringMVC配置文件:

Springmvc 默认加载 **WEB-INF/[前端控制器的名字]-servlet.xml**，也可以在前端控制器定义处指定加载的配置文件，如下

```
<init-param>
    <param-name>contextConfigLocation</param-name>
    <param-value>classpath:springmvc-servlet.xml</param-value>
</init-param>
```
如上代码，通过`contextConfigLocation`加载classpath下的`springmvc-servlet.xml`配置文件，配置文件名称可以不限定`[前端控制器的名字]-servlet.xml`。

#### 5.5 配置处理器映射器:

在`springmvc-servlet.xml`文件配置如下：

```
<!-- 根据bean的name进行查找Handler 将action的url配置在bean的name中 -->
<bean class="org.springframework.web.servlet.handler.BeanNameUrlHandlerMapping" />
```

`BeanNameUrlHandlerMapping`：表示将定义的Bean名字作为请求的url，需要将编写的controller在spring容器中进行配置，且指定bean的name为请求的url，且必须以`.action`结尾

#### 5.6 配置处理器适配器:

在`springmvc-servlet.xml`文件配置如下：

```
<bean class="org.springframework.web.servlet.mvc.SimpleControllerHandlerAdapter"/>
```

`SimpleControllerHandlerAdapter`：即简单控制器处理适配器，所有实现了`org.springframework.web.servlet.mvc.Controller` 接口的Bean作为Springmvc的后端控制器

#### 5.7 配置视图解析器:

在`springmvc-servlet.xml`文件配置如下：

```
<!-- ViewResolver -->
<bean class="org.springframework.web.servlet.view.InternalResourceViewResolver">
    <property name="viewClass" value="org.springframework.web.servlet.view.JstlView"/>
    <property name="prefix" value="/WEB-INF/jsp/"/>
    <property name="suffix" value=".jsp"/>
</bean>
```

`InternalResourceViewResolver`：支持JSP视图解析.

`viewClass`：`JstlView`表示JSP模板页面需要使用JSTL标签库，所以classpath必须包含jstl的相关jar包；

`prefix` 和 `suffix` ：查找视图页面的前缀和后缀，最终视图的址为：**前缀+逻辑视图名+后缀**，逻辑视图名需要在controller中返回`ModelAndView`指定，比如逻辑视图名为hello，则最终返回的jsp视图地址`WEB-INF/jsp/hello.jsp`.


#### 5.8 后端控制器开发:

后端控制器即`controller`，也有称为`action`。

```
importjavax.servlet.http.HttpServletRequest;
importjavax.servlet.http.HttpServletResponse;
importorg.springframework.web.servlet.ModelAndView;
importorg.springframework.web.servlet.mvc.Controller;

public class HelloWorldController implements Controller {
    @Override
    Public ModelAndView handleRequest(HttpServletRequest req, HttpServletResponse resp) throws Exception {
        ModelAndView mv = new ModelAndView();
        // 添加模型数据
        mv.addObject("message", "Hello World!");
        // 设置逻辑视图名,最终视图地址=前缀+逻辑视图名+后缀
        mv.setViewName("hello");
        return mv;
      }
}
```

`org.springframework.web.servlet.mvc.Controller`：处理器必须实现Controller 接口。
`ModelAndView`：包含了模型数据及逻辑视图名

#### 5.9 后端控制器配置:

在`springmvc-servlet.xml`文件配置如下：

```
<bean name="/hello.action" class="springmvc.action.HelloWorldController"/>
```

`name="/hello.action"` : 前边配置的`BeanNameUrlHandlerMapping`，表示如过请求的URL为`上下文/hello.action`，则将会交给该Bean进行处理。


#### 5.10 视图开发:

创建 `WEB-INF/jsp/hello.jsp`

```
<%@ page language="java" contentType="text/html; charset=UTF-8"
    pageEncoding="UTF-8"%>
<!DOCTYPE html PUBLIC "-//W3C//DTD HTML 4.01 Transitional//EN" "http://www.w3.org/TR/html4/loose.dtd">
<html>
    <head>
        <meta http-equiv="Content-Type" content="text/html; charset=UTF-8">
        <title>第一个程序</title>
    </head>
    <body>
        <!-- 显示一行信息hello world!!!! -->
        ${message}
    </body>
</html>
```

`${message}`：表示显示由HelloWorldController处理器传过来的模型数据。


#### 5.11 部署在tomcat测试

通过请求：`http://localhost:8080/springmvc_01/hello.action`, 如果页面输出“Hello World! ”就表明我们成功了。

> 总结：

主要进行如下操作：


1. 前端控制器DispatcherServlet配置
2. 加载springmvc的配置文件
3. HandlerMapping配置
4. HandlerAdapter配置
5. ViewResolver配置
6. 前缀和后缀
7. 后端控制器编写
8. 后端控制器配置
9. 视图编写


从上边的步骤可以看出，通常情况下我们只需要编写后端控制器和视图。

### 6.HandlerMapping处理器映射器

`HandlerMapping` 给前端控制器返回一个 `HandlerExecutionChain` 对象（包含一个Handler (后端控制器)对象、多个HandlerInterceptor 拦截器）对象。

#### BeanNameUrlHandlerMapping

beanName Url映射器

```
<!—beanName Url映射器 -->
<bean class="org.springframework.web.servlet.handler.BeanNameUrlHandlerMapping"/>
```

将后端控制器的bean name作为请求的url。

`SimpleUrlHandlerMapping`

```
<!—简单url映射 -->
<bean class="org.springframework.web.servlet.handler.SimpleUrlHandlerMapping">
    <property name="mappings">
        <props>
            <prop key="/hello1.action">hello_controller</prop>
            <prop key="/hello2.action">hello_controller</prop>
        </props>
    </property>
</bean>
```

可定义多个url映射至一个后端控制器，hello_controller为后端控制器bean的id。


### 7.HandlerAdapter处理器适配器

`HandlerAdapter` 会把后端控制器包装为一个适配器，支持多种类型的控制器开发，这里使用了适配器设计模式。

#### SimpleControllerHandlerAdapter

单控制器处理器适配器.

所有实现了`org.springframework.web.servlet.mvc.Controller`接口的Bean作为Springmvc的后端控制器。

适配器配置如下：

```
<bean class="org.springframework.web.servlet.mvc.SimpleControllerHandlerAdapter" />
```

#### HttpRequestHandlerAdapter

HTTP请求处理器适配器.

HTTP请求处理器适配器将`http`请求封装成`HttpServletResquest`和`HttpServletResponse`对象，和servlet接口类似。

适配器配置如下：

```
<bean class="org.springframework.web.servlet.mvc.HttpRequestHandlerAdapter"/>
```

Controller实现如下：

```
publicclass HelloWorldController2 implements HttpRequestHandler {

    @Override
    publicvoid handleRequest(HttpServletRequest request,HttpServletResponse response) throws ServletException, IOException {
        request.setAttribute("message", "HelloWorld!");
        request.getRequestDispatcher("/WEB-INF/jsp/hello.jsp").forward(request, response);
            // 也可以自定义响应内容
            //response.setCharacterEncoding("utf-8");
            //response.getWriter().write("HelloWorld!");
    }
}
```

从上边可以看出此适配器器的`controller`方法没有返回`ModelAndView`，可通过response修改定义响应内容。

### Controller控制器

**AbstractCommandController(命令控制器)**

控制器能把请求参数封装到一个命令对象(模型对象)中。

```
public class MyCommandController extends AbstractCommandController {

    /**
     * 通过构造函数设置命令对象
     */
    public MyCommandController(){
        this.setCommandClass(Student.class);
        this.setCommandName("student");//不是必须设置
    }

    @Override
    protected ModelAndView handle(HttpServletRequest request,
            HttpServletResponse response, Object command, BindException errors)
            throws Exception {

        Student student = (Student)command;
        System.out.println(student);
        Return null;
    }

    /**
     * 字符串转换为日期属性编辑器
     */
    @Override
    Protected void initBinder(HttpServletRequest request,
            ServletRequestDataBinder binder) throws Exception {
        binder.registerCustomEditor(Date.class, new CustomDateEditor(new SimpleDateFormat("yyyy-MM-dd"),true));
    }

}
```

```
publicclass Student {

    public Student() {
    }
    public Student(String name) {
        this.name = name;
    }
    private String name;//姓名
    private Integer age;//年龄
    private Date birthday;//生日
    private String address;//地址

….get/set方法省略
```

Controller配置


```
<bean id="command_controller" name="/command.action" class="springmvc.action.MyCommandController"/>
```

使用命令控制器完成查询列表及表单提交

代码略


### 9.问题解决:


#### 日期格式化

##### 在controller注册属性编辑器:

```
/**
 * 注册属性编辑器(字符串转换为日期)
 */
@InitBinder
public void initBinder(HttpServletRequest request,ServletRequestDataBinder binder) throws Exception {
    binder.registerCustomEditor(Date.class, new CustomDateEditor(new SimpleDateFormat("yyyy-MM-dd"),true));
}
```

#### Post时中文乱码

在`web.xml`中加入：

```
<filter>
    <filter-name>CharacterEncodingFilter</filter-name>
    <filter-class>org.springframework.web.filter.CharacterEncodingFilter</filter-class>
    <init-param>
        <param-name>encoding</param-name>
        <param-value>utf-8</param-value>
    </init-param>
</filter>
<filter-mapping>
    <filter-name>CharacterEncodingFilter</filter-name>
    <url-pattern>/*</url-pattern>
</filter-mapping>
```

以上可以解决post请求乱码问题。

对于get请求中文参数出现乱码解决方法有两个：

修改tomcat配置文件添加编码与工程编码一致，如下：

```
<Connector URIEncoding="utf-8" connectionTimeout="20000" port="8080" protocol="HTTP/1.1" redirectPort="8443"/>
```

另外一种方法对参数进行重新编码：

```
String userName new String(request.getParamter("userName").getBytes("ISO8859-1"),"utf-8")
```

ISO8859-1是tomcat默认编码，需要将tomcat编码后的内容按utf-8编码

###  10.注解开发:

#### 第一个例子

创建工程的步骤同第一个`springmvc`工程，注解开发需要修改`handlermapper`和`handlMapperAdapter`，如下：

在springmvc-servlet.xml中配置：

```
<!--注解映射器 -->   
<bean class="org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerMapping"/>
<!--注解适配器 -->
<bean class="org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerAdapter"/>
```
注解映射器和注解适配器可以使用 `<mvc:annotation-driven />` 代替。
`<mvc:annotation-driven />` 默认注册了注解映射器和注解适配器等bean。

#### HelloWorldController编写：

```
@Controller
public class HelloWorldController {

    @RequestMapping(value="/hello")
    public String hello(Model model)throws Exception{
        model.addAttribute("message", "HelloWorld!");
        return"hello";
    }
}
```
**注解描述**：

`@Controller`：用于标识是处理器类
`@RequestMapping`：请求到处理器功能方法的映射规则

**Controller配置**

在`springmvc-servlet.xml`中配置定义的controller：

```
<!-- controller -->
<bean class="springmvc.action.HelloWorldController"/>
```

**组件扫描:**

```
<context:component-scan base-package="springmvc.action" />
```


扫描 `@component` 、`@controller`、`@service`、`@repository` 的注解

注意：如果使用组件扫描则`controller`不需要在`springmvc-servlet.xml`中配置

### 12.@Controller

标识该类为控制器类，`@controller`、`@service`、`@repository` 分别对应了web应用三层架构的组件即控制器、服务接口、数据访问接口。

### 13.@RequestMapping

#### URL路径映射:

```
@RequestMapping(value="/user")或@RequestMapping("/user")
```

#### 根路径+子路径

根路径： `@RequestMapping` 放在类名上边，如下：

```
@Controller
@RequestMapping("/user")
```

#### 子路径：

`@RequestMapping` 放在方法名上边，如下：

```
@RequestMapping("/useradd")
public String useradd(….
```

#### URI 模板模式映射


`@RequestMapping(value="/useredit/{userId}")`：`{×××}`占位符，请求的URL可以是`/useredit/001`或`/useredit/abc`，
通过在方法中使用`@PathVariable` 获取`{×××}`中的`×××`变量。

```
@RequestMapping("/useredit/{userid}")
public String useredit(@PathVariable String userid,Model model) throws Exception{
    //方法中使用@PathVariable获取useried的值，使用model传回页面
    model.addAttribute("userid", userid);
    return"/user/useredit";
}
```

实现restFul,所有的url都是一个资源的链接，有利于搜索引擎对网址收录。


#### 多个占位符：

```
@RequestMapping("/useredit/{groupid}/{userid}")
public String useredit(@PathVariable String groupid,@PathVariable String userid,Model model) throws Exception{
    //方法中使用@PathVariable获取useried的值，使用model传回页面
    model.addAttribute("groupid", groupid);
    model.addAttribute("userid", userid);
    return"/user/useredit";
}
```

**页面**

```
<td>
    <a href="${pageContext.request.contextPath}/user/editStudent/${student.id}/${student.groupId}/${student.sex}.action">修改</a>
</td>
```

####  请求方法限定:

- 限定GET方法  `@RequestMapping(method = RequestMethod.GET)`
- 限定POST方法 `@RequestMapping(method = RequestMethod.POST)`
- Get POST 都可以 `@RequestMapping(method={RequestMethod.GET,RequestMethod.POST})`

### 14-请求数据绑定:

Controller方法通过形参接收页面传递的参数。

```
@RequestMapping("/userlist")
public String userlist(
HttpServletRequest request,
            HttpServletResponse response,
            HttpSession session,
            Model model
)
```

> 默认支持的参数类型:

**HttpServletRequest** :  通过request对象获取请求信息

**HttpServletResponse** : 通过response处理响应信息

**HttpSession** : 通过session对象得到session中存放的对象
```
session.setAttribute("userinfo", student);
session.invalidate(); // 使session失效
```

**Model** : `通过model向页面传递数据，如下：`

```
model.addAttribute("user", new User("李四"));
```

页面通过${user.XXXX}获取user对象的属性值。

#### @RequestParam绑定单个请求参数

`value`：参数名字，即入参的请求参数名字，如value=“studentid”表示请求的参数区中的名字为studentid的参数的值将传入；

`required`：是否必须，默认是true，表示请求中一定要有相应的参数，否则将报400错误码；
`defaultValue`：默认值，表示如果请求中没有同名参数时的默认值

定义如下：

```
public String userlist(@RequestParam(defaultValue="2",value="group",required=true) String groupid) {

}
```

#### @PathVariable 绑定URI 模板变量值

@PathVariable用于将请求URL中的模板变量映射到功能处理方法的参数上

```
@RequestMapping(value="/useredit/{groupid}/{userid}",method={RequestMethod.GET,RequestMethod.POST})
public String useredit(@PathVariable String groupid,@PathVariable String userid,Model model) throws Exception{
    //方法中使用@PathVariable获取useried的值，使用model传回页面
    model.addAttribute("groupid", groupid);
    model.addAttribute("userid", userid);
    return"/user/useredit";
}
```

#### @RequestBody

作用：

`@RequestBody`注解用于读取http请求的内容(字符串)，通过springmvc提供的HttpMessageConverter接口将读到的内容转换为json、xml等格式的数据并绑定到controller方法的参数上。

本例子应用：

`@RequestBody`注解实现接收http请求的json数据，将json数据转换为java对象.

#### @ResponseBody

作用：

该注解用于将Controller的方法返回的对象，通过`HttpMessageConverter`接口转换为指定格式的数据如：json,xml等，通过Response响应给客户端

本例子应用：

`@ResponseBody`注解实现将controller方法返回对象转换为json响应给客户端

> 请求json，响应json实现：

页面上请求的是一个json串，页面上实现时需要拼接一个json串提交到action。

Action将请求的json串转为java对象。SpringMVC利用@ResquesBody注解实现.

Action将java对象转为json，输出到页面。SpringMVC利用@ResponseBody注解实现.

- 环境准备

`Springmvc`默认用`MappingJacksonHttpMessageConverter`对json数据进行转换，需要加入jackson的包，如下：

- 配置:

在注解适配器中加入`messageConverters`

```
<bean class="org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerAdapter">
    <property name="messageConverters">
        <list>
            <bean class="org.springframework.http.converter.json.MappingJacksonHttpMessageConverter"></bean>
        </list>
    </property>
</bean>
```
- controller编写

```
/**
 * 请求json单个对象，返回json单个对象
 * @param user
 * @return
 * @throws Exception
 */
@RequestMapping("/requestjson")
//@RequestBody接收json串自动转为user对象，@ResponseBody将user转为json数据响应给客户端
Public @ResponseBody User requestjson(@RequestBody User user)throws Exception{
    System.out.println(user);
    return user;
}
```

- 页面js方法编写：

```
function request_json(){
       //将json对象传成字符串
  var user = JSON.stringify({name: "张三", age: 3});
  alert(user);
    $.ajax(
      {
          type:'post',
          url:'${pageContext.request.contextPath}/requestjson.action',
          contentType:'application/json;charset=utf-8',//请求内容为json
          data:user,
          success: function(data){
              alert(data.name);
          }
      }   
  )
}
```

### 15-结果转发:

#### Redirect

Contrller方法返回结果重定向到一个url地址，如果方式：

```
return "redirect:/user/userlist.action";
```

`redirect`方式相当于`response.sendRedirect()`，转发后浏览器的地址栏变为转发后的地址，因为转发即执行了一个新的request和response。
由于新发起一个request原来的参数在转发时就不能传递到下一个url，如果要传参数可以`/user/userlist.action`后边加参数，如下：

```
/user/userlist.action?groupid=2&…..
```

#### forward

controller方法执行后继续执行另一个controller方法。

```
return "forward:/user/userlist.action";
```

`forward` 方式相当于`request.getRequestDispatcher().forward(request,response)`，转发后浏览器地址栏还是原来的地址。转发并没有执行新的request和response，而是和转发前的请求共用一个request和response。所以转发前请求的参数在转发后仍然可以读取到。

```
@RequestMapping("/c")
public String c(String groupid,UserVo userVo)throws Exception{

    System.out.println("...c...."+groupid+"...user..."+userVo.getUser());
    return "forward:/to/d.action";
}

@RequestMapping("/d")
public String d(String groupid,UserVo userVo)throws Exception{

    System.out.println("...d...."+groupid+"...user..."+userVo.getUser());
    return "success";
}
```



### 16-拦截器

#### 定义

Spring Web MVC 的处理器拦截器类似于Servlet 开发中的过滤器Filter，用于对处理器进行预处理和后处理。
SpringMVC拦截器是针对mapping配置的拦截器

#### 拦截器定义

实现`HandlerInterceptor`接口，如下：

```
Public class HandlerInterceptor1 implements HandlerInterceptor{

    /**
     * controller执行前调用此方法
     * 返回true表示继续执行，返回false中止执行
     * 这里可以加入登录校验、权限拦截等
     */
    @Override
    Public boolean preHandle(HttpServletRequest request,
            HttpServletResponse response, Object handler) throws Exception {
        Return false;
    }
    /**
     * controller执行后但未返回视图前调用此方法
     * 这里可在返回用户前对模型数据进行加工处理，比如这里加入公用信息以便页面显示
     */
    @Override
    Public void postHandle(HttpServletRequest request,
            HttpServletResponse response, Object handler,
            ModelAndView modelAndView) throws Exception {
    }
    /**
     * controller执行后且视图返回后调用此方法
     * 这里可得到执行controller时的异常信息
     * 这里可记录操作日志，资源清理等
     */
    @Override
    Public void afterCompletion(HttpServletRequest request,
            HttpServletResponse response, Object handler, Exception ex)
            throws Exception {      
    }
}
```

#### 拦截器配置

针对某种mapping 配置拦截器

```
<bean
    class="org.springframework.web.servlet.handler.BeanNameUrlHandlerMapping">
    <property name="interceptors">
        <list>
            <ref bean="handlerInterceptor1"/>
            <ref bean="handlerInterceptor2"/>
        </list>
    </property>
</bean>
<bean id="handlerInterceptor1" class="springmvc.intercapter.HandlerInterceptor1"/>
<bean id="handlerInterceptor2" class="springmvc.intercapter.HandlerInterceptor2"/>
```


针对所有mapping配置全局拦截器

```
<!--拦截器 -->
<mvc:interceptors>
    <!--多个拦截器,顺序执行 -->
    <mvc:interceptor>
        <mvc:mapping path="/**"/>
        <bean class="cn.itcast.springmvc.filter.HandlerInterceptor1"></bean>
    </mvc:interceptor>
    <mvc:interceptor>
        <mvc:mapping path="/**"/>
        <bean class="cn.itcast.springmvc.filter.HandlerInterceptor2"></bean>
    </mvc:interceptor>
</mvc:interceptors>
```

#### 拦截器应用

用户身份认证

```
Public class LoginInterceptor implements HandlerInterceptor{

    @Override
    Public boolean preHandle(HttpServletRequest request,
            HttpServletResponse response, Object handler) throws Exception {

        //如果是登录页面则放行
        if(request.getRequestURI().indexOf("login.action")>=0){
            return true;
        }
        HttpSession session = request.getSession();
        //如果用户已登录也放行
        if(session.getAttribute("user")!=null){
            return true;
        }
        //用户没有登录挑战到登录页面
        request.getRequestDispatcher("/WEB-INF/jsp/login.jsp").forward(request, response);

        return false;
    }
}
```
