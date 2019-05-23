---
layout:     post
title:      Spring | Bean之注解配置形式
subtitle:   Bean篇(2)注解配置形式
date:       2018-11-20
author:     Aiden
header-img: img/post-bg-swift2.jpg
catalog: true 			
tags:								
    - Spring
---

使用注解配置的形式来构造bean需要首先引入 `context:component-scan` 的配置。他指定了 spring ioc 需要扫描的包，包下面的类将通过注解配置，构造bean对象。

例如:

```
<context:component-scan base-package="com.saligia.spring.bean.annotation"></context:component-scan>
```

### @Configuration 组件


这种配置与在xml传参构造 bean 对象目的是一致的, IOC 容器初始化时候会扫描 `@Configuration` 注解的类，并扫描这个类下所有的 `@Bean` 注解的方法，并根据配置调用拿到 bean， 装载到容器中。

比如我们构造 `Persion`类, 我们使用 `@Value` 属性来设置bean字段默认值 :

```
public class Persion {
    private String name;
    private int value;

    @Value("saligia")
    public void setName(String name) {
        this.name = name;
    }

    @Value("30")
    public void setValue(int value) {
        this.value = value;
    }

    @Override
    public String toString() {
        return "Persion{" +
                "name='" + name + '\'' +
                ", value='" + value + '\'' +
                '}';
    }
}
```

我们这时候通过注解形式构造这个 bean,看一下 :

```
@Configuration
public class BeanConfiguraton {

    @Bean("persion")
    public Persion getPersion(){
        Persion persion = new Persion();
        return persion;
    }
}
```

这个时候我们就可以通过 `Persion persion = (Persion)ioc.getBean("persion");` 的方式拿到这个bean对象。


#### @Configuration 使用 properties 注入变量

使用外置的 `properties` 外部属性文件配置的时候需要加入一个配置依赖。`location` 属性指定属性文件位置。

```
<context:property-placeholder location="classpath:system.properties"></context:property-placeholder>
```

例如,我们的属性文件内容为:

```
sys.name:saligia
sys.value:20
```

我们将 `Persion` 中的 `@Value` 修改为使用占位符来表示即可 :

```
@Value("${sys.name}")
@Value("${sys.value}")
```

#### 自动装配:

当我们字段依赖于其他 bean 时候， 我们可以通过 bean 的自动装配功能来实现。

`@Autowired` 注解可以帮助我们实现自动装配的功能， 注解可以用来修饰 **变量名** 跟 **setter** 方法。
这时，**容器将扫描所有的 bean, 找一个类型匹配的bean自动注入到修饰变量中**。

```
public class Persion {
    private String name;
    private int value;
    private Address address;

    @Value("saligia")
    public void setName(String name) {
        this.name = name;
    }

    @Value("30")
    public void setValue(int value) {
        this.value = value;
    }

    @Autowired
    public void setAddress(Address address){
        this.address = address;
    }
}
```

如果 ioc 容器匹配到了多个相同类型的bean, 如下所示, 则注入过程会报失败。

```
@Configuration
public class BeanConfiguraton {

    @Bean("persion")
    public Persion getPersion(){
        Persion persion = new Persion();
        return persion;
    }

    @Bean("address1")
    public Address getAddress1(){
        return new Address("address1");
    }

    @Bean("address2")
    public Address getAddress2(){
        return new Address("address2");
    }
}
```
这个时候我们就需要借助 `@Qualifier` 注解， 这将使用 setBean 的 **byName** 方式。

```
@Autowired
@Qualifier("address1")
public void setAddress(Address address){
    this.address = address;
}
```

如果没有启用 `@Qualifier` 的多个相同类型的Bean的场景， 我们可以使用 `@Primary` 属性来为bean 指定一个首选着。

```
@Bean("address1")
@Primary
public Address getAddress1(){
    return new Address("address1");
}

@Bean("address2")
public Address getAddress2(){
    return new Address("address2");
}
```
#### 特殊的 Bean :

- **@Component**: 通用的构造型注解，标识该类为spring组件[不推荐使用]
- **@Controller**: 标识将该类定义为SpringMVC的控制器 ==> Controller层
- **@Service**: 标识将该类定义为业务服务==> Service层
- **@Repository**: 标识将该类定义为数据仓库==> Dao层

目前4种注解意思是一样，并没有什么区别，区别只是名字不同。使用方法：

```
@Component
public class Message {
    private String message ;
    @Value("testMessage")
    public void setMessage(String message) {
        this.message = message;
    }
    @Override
    public String toString() {
        return message;
    }
}
```

使用这个注解将自动将这个类加入 IOC容器中。我们也可以为 Bean 这是Name 如:`@Component("message")`


#### Bean 初始化与释放

实现初始化和销毁bean之前进行的操作，只能有一个方法可以用此注释进行注释，方法不能有参数，返回值必需是void,方法需要是非静态的。
例如:

```
@Component("message")
public class Message {
    private String message ;

    @PostConstruct
    public void init(){
        System.out.println("init Message");
    }
    @Value("testMessage")
    public void setMessage(String message) {
        this.message = message;
    }
    @PreDestroy
    public void destroy(){
        System.out.println("destroy Message");
    }
}
```

#### 懒加载(@Lazy(true))

用于指定该Bean是否取消预初始化，用于注解类，延迟初始化。
