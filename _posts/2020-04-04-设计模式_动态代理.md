---
title: 设计模式--动态代理
data: 2020-04-04
category: 设计模式
tags: Jdk Cglib
description: 代理模式是23种设计模式的一种，他是指一个对象A通过持有另一个对象B，可以具有B同样行为的模式。静态代理，是在编译期就生成被代理对象的代理实例，开发阶段代码极度冗余，且不便于扩展。动态代理，是在运行期动态生成被代理对象的代理实例，扩展性很高且应用广泛。此文分析了两种常用的动态代理模式：JDK动态代理和cglib动态代理。
---

动态代理有什么用？

在不改动服务类方法实现的基础上，增强服务类方法的功能，如：调用服务类方法前，打印方法入参日志。

## JDK动态代理

原理：目标服务类JdkExampleImpl实现目标接口JdkExample，运行期间，动态生成目标接口JdkExample的另一实现类$Proxy0的实例，$Proxy0即为JdkExampleImpl的代理对象，通过调用$Proxy0的methodOne方法实现调用JdkExampleImpl的methodOne方法。

1）**必须有一个目标接口服务JdkExample**，包含methodOne功能

```java
public interface JdkExample {
    void methodOne();
}
```

2）接口服务实现类JdkExampleImpl

```java
public class JdkExampleImpl implements JdkExample {
    @Override
    public void methodOne() {
        System.out.println("method one");
    }
}
```

3）代理调用器**InvocationHandler**（非代理对象），内部包装了一个JdkExampleImpl实例。

invoke()为增强后的方法，System.out.println("extend function")为增强功能，method.invoke(target, args)为调用目标对象方法

```java
public class JdkProxy implements InvocationHandler {
    // 目标类，也就是被代理对象
    private Object target;

    public JdkProxy(Object target) {
        this.target = target;
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        // 此处为增强功能
        System.out.println("extend function");

        // 调用target的method方法
        return method.invoke(target, args);
    }
}
```

4）编写测试类

```java
public class JDKMain {
    public static void main(String[] args) {
        Object proxyInstance = Proxy.newProxyInstance(
                JdkExampleImpl.class.getClassLoader(),
                JdkExampleImpl.class.getInterfaces(),
                new JdkProxy(new JdkExampleImpl()));

        // proxyJdkExample的类型是实现了JdkExample接口的新类型"$Proxy+数字"
        JdkExample proxyJdkExample = (JdkExample) proxyInstance;
        System.out.println("proxyJdkExample: " + proxyJdkExample.getClass().getName());
        proxyJdkExample.methodOne();
    }
}
```

5）运行测试类，查看结果

```sh
proxyJdkExample: com.sun.proxy.$Proxy0
extend function
method one
```

## cglib动态代理

原理：和JDK动态代理一样，运行期间动态生成一个目标对象的代理类，通过调用代理类的方法达到调用目标对象的方法。不同的是：JDK动态代理必须实现接口，运行期间生成目标接口的另一个实现类；而cglib动态代理不需要实现接口，运行期间生成目标对象的子类（即代理对象）。

1）目标服务类CglibExample，包含methodOne功能

```java
public class CglibExample {
    public void methodOne() {
        System.out.println("method one");
    }
}
```

2）代理拦截器**MethodInterceptor**（非代理对象）

```java
public class CglibProxy implements MethodInterceptor {
    @Override
    public Object intercept(Object proxy, Method method, Object[] objects, MethodProxy methodProxy) throws Throwable {
        // 此处为增强功能
        System.out.println("extend function");
        return methodProxy.invokeSuper(proxy, objects);
    }
}
```

3）编写测试类

```java
public class CglibMain {
    public static void main(String[] args) {
        Enhancer enhancer = new Enhancer();
        enhancer.setSuperclass(CglibExample.class);
        enhancer.setCallback(new CglibProxy());
        Object object = enhancer.create();

        // proxyCglibExample继承CglibExample,类型"CglibExample$$EnhancerByCGLIB$$+字符串"
        CglibExample proxyCglibExample = (CglibExample) object;
        System.out.println("cglibExample: " + proxyCglibExample.getClass().getName());
        proxyCglibExample.methodOne();
    }
}
```

4）运行测试类，查看结果

```sh
cglibExample: cglib.CglibExample$$EnhancerByCGLIB$$d9e4c56
extend function
method one
```

## 写在最后

1）jdk创建对象的速度远大于cglib，这是由于cglib创建对象时需要操作字节码。cglib执行速度略大于jdk，所以比较适合单例模式。另外由于CGLIB的大部分类是直接对Java字节码进行操作，这样生成的类会在Java的永久堆中。如果动态代理操作过多，容易造成永久堆满，触发OutOfMemory异常。spring默认使用jdk动态代理，如果类没有接口，则使用cglib。

2）动态代理应用非常广泛，spring的ioc依赖注入、aop切面编程，osgi的service注册，odl的rpc注册 ...

[源码demo地址](https://github.com/leihuan7086/design-pattern-demo)