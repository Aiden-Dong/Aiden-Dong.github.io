---
layout:     post
title:      Spring | AOP之基础之动态代理的实现
subtitle:   AOP篇(1)基础之动态代理的实现
date:       2018-11-26
author:     Aiden
header-img: img/post-bg-swift2.jpg
catalog: true 			
tags:								
    - Spring
---

#### 介绍

**Spring AOP** 主要通过 **动态代理** 来实现的，所以我们需要在介绍 AOP 用法之前，先来介绍下动态代理的用法以及本质。

对于动态代理的理解可以借鉴普通**代理模式**。我们普通的Java代理需要为一个对象建立专门的代理对象，通过调用代理对象，来实现原对象的各种功能。

![image.png]({{ site.url }}/assets/spring_3_1.png)

而动态代理区别于普通代理方式,动态代理是在运行时产生的代理类。

---

#### 普通代理的实现方式：

**接口 :**

```
public interface Subject {
    public void request();
}
```

**实现类:**

```
public class RealSubject implements Subject{
    @Override
    public void request() {
        System.out.println("request");
    }
}
```

**代理类:**

```
public class RealProxySubject implements Subject {

    private Subject subject;

    public RealProxySubject(){
        this.subject = new RealSubject();
    }

    public void preRequest(){
        System.out.println("Preset Request");
    }

    public void afterRequest(){
        System.out.println("After Request");
    }

    @Override
    public void request() {
        this.preRequest();
        subject.request();
        this.afterRequest();
    }
}
```

**调用方式:**

```
public static void main(String[] args) {
    Subject subject = new RealProxySubject();
    subject.request();
}
```

这里我们重新再看下普通代理类，来跟动态代理做下比较。 首先必须要定义接口(`Subject`), 实现类(`RealSubject`)与代理类(`RealProxySubject`)都必须要继承接口。

关于实现类 `RealSubject` 要暴露main方法中， 还是要在代理类中构造，这个可以根据具体的场景合理选择。

#### 动态代理实现方式

接下来我们看下动态代理与普通代理方式的区别与联系， 动态代理需要使用的 Java 的反射相关的包来实现：

- 代理的实现细节类需要继承 `java.lang.reflect.InvocationHandler` 接口, 在 `invoke` 方法中实现代理细节；
- 通过 `java.lang.reflect.Proxy` 类指定 ClassLoader 对象和一组 interface, 与代理的实现细节类来创建动态代理类；

我们看下动态代理的实现细节:

- **接口**:

```
public interface Subject {
    public int request();
}
```

- **实现类**:

```
public class RealSubject implements Subject{
    @Override
    public int request() {
        System.out.println("hello world");
        return 10;
    }
}
```

- **代理的实现细节类** :

```
public class SubjectProxy implements InvocationHandler {

    private Subject subject;

    public SubjectProxy(Subject subject) {
        this.subject = subject;
    }

    public Subject getSubject(){
        return (Subject) Proxy.newProxyInstance(subject.getClass().getClassLoader(), subject.getClass().getInterfaces(), this);
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        System.out.println("Preset Action");
        Object result= method.invoke(subject, args);
        System.out.println("After Action");
        return result;
    }
}
```

- **使用方式**：

```
public static void main(String[] args) {
  SubjectProxy proxy = new SubjectProxy(new RealSubject());
  Subject subject = proxy.getSubject();
  int result = subject.request();
  System.out.println(result);
}
```

> **说明**:

从上面的例子中可以看出， 动态代理的重点部分就是这个**代理的实现细节类(SubjectProxy)**, **代理的实现细节类(SubjectProxy)** 负责实现代理细节 (通过`invoke`方法)。并构造一个匿名的代理类(`Proxy.newProxyInstance` 方法)。

`Proxy.newProxyInstance` 通过类加载器与指定的接口，构建一个继承匿名的代理类：

```
public final class $Proxy0 extends Proxy implements Subject {
  private static Method m1;
  private static Method m3;
  private static Method m0;
  private static Method m2;

  public $Proxy0 (InvocationHandler paramInvocationHandler) {
    super(paramInvocationHandler);
  }

  public final boolean equals(Object paramObject) {
    try {
      return ((Boolean)this.h.invoke(this, m1, new Object[] { paramObject })).booleanValue();
    }
    catch (Error|RuntimeException localError) {
      throw localError;
    }
    catch (Throwable localThrowable) {
      throw new UndeclaredThrowableException(localThrowable);
    }
  }

  public final void request() {
    try {
      this.h.invoke(this, m3, null);
      return;
    }
    catch (Error|RuntimeException localError) {
      throw localError;
    }
    catch (Throwable localThrowable) {
      throw new UndeclaredThrowableException(localThrowable);
    }
  }

  public final int hashCode() {
    try {
      return ((Integer)this.h.invoke(this, m0, null)).intValue();
    }
    catch (Error|RuntimeException localError) {
      throw localError;
    }
    catch (Throwable localThrowable) {
      throw new UndeclaredThrowableException(localThrowable);
    }
  }

  public final String toString() {
    try {
      return (String)this.h.invoke(this, m2, null);
    }
    catch (Error|RuntimeException localError) {
      throw localError;
    }
    catch (Throwable localThrowable) {
      throw new UndeclaredThrowableException(localThrowable);
    }
  }

  static {
    try {
      m1 = Class.forName("java.lang.Object").getMethod("equals", new Class[] { Class.forName("java.lang.Object") });
      m3 = Class.forName("com.saligia.spring.aop.proxy.dynamic.Subject").getMethod("request", null);
      m0 = Class.forName("java.lang.Object").getMethod("hashCode", new Class[0]);
      m2 = Class.forName("java.lang.Object").getMethod("toString", new Class[0]);
      return;
    }
    catch (NoSuchMethodException localNoSuchMethodException) {
      throw new NoSuchMethodError(localNoSuchMethodException.getMessage());
    }
    catch (ClassNotFoundException localClassNotFoundException) {
      throw new NoClassDefFoundError(localClassNotFoundException.getMessage());
    }
  }
}
```

从上面的可以看出来, 匿名类继承了 **Proxy**类与**Subject**接口， 当调用了 subject 中的方法时， 会转为我们在**SubjectProxy**重写的 `invoke` 方法。

所以，我们这里总结下:

`Proxy.newProxyInstance` 返回的对象会根据，类加载器，接口，还有代理构建一个与 **接口** 跟 **InvocationHandler实现** 相关的匿名代理类， 对于代理类本身来说， 至于接口相关，跟接口具体实现的哪个具体子类并无关系。

而产生关系来自于 **InvocationHandler 实现**, 他将接口方法使用反射形式传递给了子类实例的相关方法的调用。

并且对于 `InvocationHandler` 的 `invoke`方法，我们通过这个匿名代理类也可以解读


```
/**
 * @param : proxy  匿名代理类
 * @param : method 通过接口解析方法
 * @param : args   匿名代理类通过方法接受的函数参数
 */
public Object invoke(Object proxy, Method method, Object[] args) throws Throwable
```

这样我们可以通过 **Proxy** 跟 `InvocationHandler` 便构造出来了我们需要的实现类的代理类。

> **总结**:

1. 从上所述可以看出,代理模式必须要有**接口**, 代理类跟实现类都是继承自同一个接口。
2. 我们只能通过接口类型去接收匿名代理类：如上面的 `Subject subject = proxy.getSubject();`
3. 匿名代理类与实现类的关系在`InvocationHandler.invoke`中实现。
4. 关于`invoke`方法中的`proxy`字段一般不要调用， 从上面可以看出，容易引起死循环。
