---
layout:     post
title:      Spring 教程 | AOP的切面编程技术 AspectJ
subtitle:   AOP篇(2)切面编程技术 AspectJ
date:       2018-12-23
author:     Aiden
header-img: img/post-bg-swift2.jpg
catalog: true 			
tags:								
    - Spring
---
### 介绍 :

AOP(Aspect Orient Programming) 既为面向切面编程。
它可以说是OOP编程的一种扩展与补充，可以较为友好的处理不同模块之间具有横向相关性质的一类问题，比如日志管理，安全机制等。

我们这里 AOP 技术用的 AspectJ 的注解形式处理的。

```
<dependency>
    <groupId>org.aspectj</groupId>
    <artifactId>aspectjweaver</artifactId>
    <version>1.8.13</version>
</dependency>
```

### 来个例子:

我们在`com.saligia.spring.aop.aspectj`下面开发我们的事例程序。

首先定义 bean 的注解配置:

```
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:p="http://www.springframework.org/schema/p" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:context="http://www.springframework.org/schema/context"
       xmlns:aop="http://www.springframework.org/schema/aop"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
     http://www.springframework.org/schema/beans/spring-beans-4.1.xsd
     http://www.springframework.org/schema/context
     http://www.springframework.org/schema/context/spring-context-4.1.xsd
     http://www.springframework.org/schema/aop http://www.springframework.org/schema/aop/spring-aop-4.1.xsd">


   <context:component-scan base-package="com.saligia.spring.aop.aspectj"></context:component-scan>
   <aop:aspectj-autoproxy/>
</beans>
```

首先定义一个我们的实际功能类 **Persion**:

```
@Component("persion")
public class Persion {
    private String name;

    @Value("saligia")
    public void setName(String name){
        this.name = name;
    }

    public void say(){
        System.out.println("hello : " + this);
    }

    @Override
    public String toString() {
        return name;
    }
}
```

我们的通常Bean 的使用方式为 ：

```
public static void main(String[] args) {
    ApplicationContext context = new ClassPathXmlApplicationContext("com/saligia/spring/aop/aspectj/aspectJ.xml");
    Persion persion = (Persion)context.getBean("persion");
    persion.say();
}
```

这个很正常， 的情况下我们得到结果：

```
hello : saligia
```

这时候我们想使用 AOP 技术，在每次调用 say() 的时候，在调用方法内部逻辑之前先做预处理动作， 这时候我们可以写一个例子:

```
@Aspect
@Component
public class LogAspect {
    @Pointcut("execution(void com.saligia.spring.aop.aspectj.Persion.say())")
    public void pointCut(){}

    @Before("pointCut()")
    public void log(){
        System.out.println("Before say : ");
    }
}
```

我们使用 `@Pointcut` 关联我们想要处理的方法 `say()`, 然后使用 `@Before` 注解标明在调用 `say()` 方法之前先调用 `log()` 方法,这样我们的调用逻辑不变， 而此时结果为 ：

```
Before say :
hello : saligia
```

这就是 AOP 技术， 不知道有没有理解。

接下来我们详细说一下细节方面

---

### 详解 :

AOP 技术就是将我们外部定义的事件(或者说是处理逻辑)，注入到常规处理逻辑中， 一般用来作为在常规的逻辑处理范围之外的日志记录，权限校验，或者其他清理工作。

#### 切点(PointCut):

切点是指我们的外部事件所要关联的正常逻辑的位置(这里是关联的方法)。

- **execution**


`execution` 用来匹配方法签名，方法签名使用全限定名，包括访问修饰符(`public/private/protected`)返回类型，包名、类名、方法名、参数，其中返回类型，包名，类名，方法，参数是必须的：

```
@Pointcut("execution(public void com.saligia.spring.aop.aspectj.Persion.say())")
```

我们可以使用模糊匹配来匹配切点:

```
@Pointcut("execution(* com.saligia.spring..*.say(..))")
```

从上面可以看出， 关于模糊匹配的符号我们可以借助`*`,`..` 两种。

`*` 用来适配一个单词，我们可以用来表示，任意返回值，任意类或者任意方法。
`..` 用来匹配多个单词， 我们可以用来表示， 中间任意包，或者多个任意参数。
当然，我们可以指名我们的参数类型。

```
@Pointcut("execution(* com.saligia.spring..*.say(java.lang.String))")
```

- **within**

`within` 用来限定连接点属于某个确定类型的类。这个切点会关联这个类下面的所有方法

```
@Pointcut("within(com.saligia.spring.aop.aspectj.Persion)")
```

如果想要匹配某个包下面的所有的类:

```
@Pointcut("within(com.saligia.spring.aop.aspectj.*)")
```

- **args**

`args` 表达式的作用是匹配指定参数类型和指定参数数量的方法，无论其类路径或者是方法名是什么。这里需要注意的是，args指定的参数必须是全路径的。

```
@Pointcut("args(java.lang.String)")
```

也可以用来匹配多个参数:

```
@Pointcut("args(java.lang.String,..,java.lang.Integer)")
```

- **@annotation**

`@annotation` 注解用来匹配使用使用了某注解的方法

```
@Pointcut("@annotation(javax.jws.WebMethod)")
```

- **@within**

`@within` 表示匹配带有指定注解的类:

```
@Pointcut("@within(org.springframework.stereotype.Repository)")
```

- **@args**

`@args` 则表示使用指定注解标注的类作为某个方法的参数时该方法将会被匹配。

```
@Pointcut("@args(com.spring.annotation.FruitAspect)")
```

- **this 和 target**

`this` 和 `target` 表达式中都只能指定类或者接口，在面向切面编程规范中:
`this` 表示匹配调用当前切点表达式所指代对象方法的对象.
`target` 表示匹配切点表达式指定类型的对象。

比如有两个类A和B，并且A调用了B的某个方法 :

如果切点表达式为this(B)，那么A的实例将会被匹配.
如果这里切点表达式为target(B)，那么B的实例也即被匹配。

```
@Pointcut("this(com.saligia.spring.aop.aspectj.Persion)")
@Pointcut("target(com.saligia.spring.aop.aspectj.Persion)")
```

- **切点表达式组合**

可以使用 `&&`、`||`、`!`、三种运算符来组合切点表达式，表示与或非的关系。

```
@Pointcut("within(com.saligia.spring.aop.aspectj.Persion) || args()")
```


#### 通知(Advice):

通知是指我们事件的具体逻辑内容(比如上面的`log()`),
以及触发的时机既在切点的是么时候触发,Spring 有 5种类型的触发时机可以使用 :

- 前置通知(@Before) : 在目标方法被调用之前调用通知功能
- 后置通知(@After) : 在方法返回或者抛出异常后调用通知。
- 返回通知(@AfterReturning) : 在目标方法被调用之后调用通知。
- 异常通知(@AfterThrowing) : 在目标方法抛出异常之后调用通知。
- 环绕通知(@Around) :通知包裹了被通知的方法,通知的方法调用之前和调用之后执行自定义的行为。

#### 切面(Aspect):

切面是切点跟通知的组合。

一个切面完整说明了一个事件触发的地点，时机，内容。

- **前置通知**

```
@Pointcut("execution(public void com.saligia.spring.aop.aspectj.Persion.say())")
public void execPointCut(){}

@Before("execPointCut()")
public void log(){
    System.out.println("Before say : ");
}
```

所以当我们调用 `persion.say();`

显示结果:

```
Before say :
hello : saligia
```

- **后置通知** :

```
@After("execPointCut()")
public void log(){
    System.out.println("After say : ");
}
```

显示结果 :

```
hello : saligia
After say :
```

- **环绕通知**:

```
@Around("execPointCut()")
public void log(ProceedingJoinPoint joinPoint){
    // 方法执行前的动作
    System.out.println("Before say : ");
    try {
        // 执行目标方法
        joinPoint.proceed();
    } catch (Throwable throwable) {
        // 异常抛出之后的动作
        System.out.println("After Exception say :");
    }finally {
        // 无论是否抛出异常都执行的动作
    }
    // 正常结束后的动作
    System.out.println("After say : ");
}
```

显示方式 :

```
Before say :
hello : saligia
After say :
```


**其他的通知方式写法与这些有雷同，在这里就不在一一表述**

#### 连接点(JoinPoint)

连接点是一个动态的概念，是指在应用程序执行过程中插入切面的位置点。
比如程序执行之前，程序异常抛出时候等等。


> 总结内容：

使用 **SpringAop** 可以帮助我们在对代码非侵入的情况下实现功能块的侵入，在系统架构保持一定的系统完整性与可拔插与复用性。

但是由于这种操作是非感知性的， 在一定程度上增加了代码阅读的复杂性。

所以有关利弊性取舍还是酌情考虑是否使用SpringAOP 还是手动编写动态代理形式。
