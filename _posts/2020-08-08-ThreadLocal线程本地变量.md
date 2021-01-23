---
title: ThreadLocal线程本地变量
date: 2020-08-08
category: SpringCloud
tags: ThreadLocal
description: ThreadLocal(线程本地变量)——高并发线程安全的一种新思路，数据在线程间隔离，只能当前线程访问。
---

## ThreadLocal是什么

ThreadLocal 是线程本地变量，在每个线程中都创建了一个 ThreadLocalMap 对象，每个线程可以访问自己内部 ThreadLocalMap 对象内的 value。

ThreadLocal本身是线程隔离的，InheritableThreadLocal继承ThreadLocal，提供了一种父子线程之间的数据共享机制。

## ThreadLocal内部关系

### 1）核心方法

```java
public class ThreadLocal<T> {
    
    // 初始化value，默认null。子类可重写
	protected T initialValue() {
        return null;
    }
    
    // 获取当前线程存储值
    public T get() {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null) {
            ThreadLocalMap.Entry e = map.getEntry(this);
            if (e != null) {
                @SuppressWarnings("unchecked")
                T result = (T)e.value;
                return result;
            }
        }
        return setInitialValue();
    }
    
    // 设置当前线程存储值
    public void set(T value) {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null)
            map.set(this, value);
        else
            createMap(t, value);
    }
    
    // 清空当前线程存储值
    public void remove()  {
         ThreadLocalMap m = getMap(Thread.currentThread());
         if (m != null)
             m.remove(this);
     }
｝
```

### 2）Thread与ThreadLocal关系

```java
public class Thread implements Runnable {
    ThreadLocal.ThreadLocalMap threadLocals = null;
}
```

```java
public class ThreadLocal<T> {
    
    static class ThreadLocalMap {
        // 索引根据key基于hash算法得到，扩容方式为2的幂次方
        private Entry[] table;
        
        static class Entry extends WeakReference<ThreadLocal<?>> {
            /** The value associated with this ThreadLocal. */
            Object value;

            // 实际上key并不是ThreadLocal本身，而是它的一个弱引用
            Entry(ThreadLocal<?> k, Object v) {
                super(k);
                value = v;
            }
        }
    }
}
```

### 3）分析一下

a）可以看到get/set/remove方法获取的都是当前线程的ThreadLocal.ThreadLocalMap实例，ThreadLocalMap内部维护了一个Entry[]数组，Entry实例封装了我们真正的数据对象value。

b）通过ThreadLocal创建的副本是存储在每个线程自己的threadLocals中。

c）ThreadLocalMap的键key为ThreadLocal对象（实际上key并不是ThreadLocal本身，而是它的一个弱引用），每个线程可以存储多个threadLocal变量。

d）threadLocal变量是线程独有，但是threadLocal变量内部保存的数据对象value可以是线程共享。

e）get之前，必须先set，否则会报空指针异常；或者重写initialValue()方法。

f）ThreadLocalMap提供了一种为ThreadLocal定制的高效实现，并且自带一种基于弱引用的垃圾清理机制。

## ThreadLocal应用场景

### 简单示例

```java
public class TestThreadLocal {
    private static final ThreadLocal<Integer> threadlocal = new ThreadLocal<>();

    public static class MyRunnable implements Runnable {
        @Override
        public void run() {
            threadlocal.set(new Random().nextInt(10));
            try {
                Thread.sleep(2000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            System.out.println(Thread.currentThread() + " : " + threadlocal.get());
        }
    }

    public static void main(String[] args) {
        MyRunnable myRunnable = new MyRunnable();
        Thread thread1 = new Thread(myRunnable);
        Thread thread2 = new Thread(myRunnable);
        thread1.start();
        thread2.start();
    }
}
```

控制台打印

```sh
Thread[Thread-0,5,main] : 7
Thread[Thread-1,5,main] : 6
```

### 典型场景

#### 1）SimpleDateFormat

SimpleDateFormat是一个线程不安全的时间工具类，所以SimpleDateFormat不能设计为线程共享static。new一个SimpleDateFormat对象开销很大，一个线程内重复创建销毁SimpleDateFormat消耗资源的同时还影响性能。

解决方案1：Java8之后可以使用线程安全的DateTimeFormatter代替SimpleDateFormat。

解决方案2：使用本文的ThreadLocal 保存SimpleDateFormat实例，每个线程独享一个SimpleDateFormat实例。

```java
public class SimpleDateUtil {
    private static ThreadLocal<DateFormat> threadLocal = new ThreadLocal<>() {
        @Override
        protected DateFormat initialValue() {
            return new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");
        }
	};

    public static String format(Date date) {
        return threadLocal.get().format(date);
    }
｝
```

#### 2）请求拦截器+用户信息

ThreadLocal 可以保存当前线程独有的信息。比如Spring项目，在请求拦截器中保存用户信息到ThreadLocal 实例，当前线程的任何业务方法中都可以通过ThreadLocal 实例获取当前用户信息了。

#### 3）错误日志分析

多线程并发下，一个请求执行过程中的错误日志信息可能会被分散打印出来。使用ThreadLocal 可以将当前线程的错误日志缓存下来，在请求的最后一次性将缓存的错误日志全部打印出来，方便开发人员查找和定位bug。

#### 4）其它场景

ThreadLocal的应用场景非常广泛，细心会发现Spring项目很多地方都有使用。

如：数据库JDBC连接、Session管理、事务管理、Aop切面编程等等。

## ThreadLocal内存泄漏

### 强引用与弱引用

强引用，不会被gc回收。

弱引用，WeakReference类，会被gc回收。可以在缓存中使用弱引用。

### GC回收机制

1）引用计数法，计数为0时可以回收。循环引用会使计数器=1 永远无法回收。
2）可达性分析法，对象到 GC Roots 没有引用链时可以回收。

### 内存泄露分析

#### 1）内存泄露原理

```java
static class ThreadLocalMap {
    private Entry[] table;

    static class Entry extends WeakReference<ThreadLocal<?>> {
        /** The value associated with this ThreadLocal. */
        Object value;

        Entry(ThreadLocal<?> k, Object v) {
            super(k);
            value = v;
        }
    }
}
```

源码可以看到ThreadLocalMap的key是弱引用，而Value是强引用。这就导致了一个问题，ThreadLocal在没有外部对象强引用时，发生GC时弱引用Key会被回收，而Value不会回收，如果创建ThreadLocal的线程一直持续运行，那么这个Entry对象中的value就有可能一直得不到回收，发生内存泄露。

#### 2）设计Entry继承WeakReference原理

1）如果threadLocalMap的Entry强引用threadLocal，就算外部对象不再强引用threadLocal，threadLocal永远不会被垃圾回收，必定会导致内存泄露。

2）自我清理机制。ThreadLocal的get、set和remove方法查找Entry为线性探测方式，弱引用断了的key会做一次清理。一个ThreadLocal在没有外部对象强引用时，发生GC时弱引用Key会被回收。当前线程之后再调用另一个ThreadLocal的get、set和remove方法都会高概率清理掉ThreadLocalMap中key为null的对象，断开value强引用，使value可以被GC回收。

### 如何避免泄漏

**显式调用remove方法**，将Entry节点和Map的引用关系移除，这样整个Entry对象在GC Roots分析后就变成不可达了，下次GC的时候就可以被回收。

## 设计一个ThreadLocal工具类

```java
public class ThreadLocalUtil {

    private static final ThreadLocal<Map<String, Object>> threadLocal = new NamedThreadLocal("xxx-threadLocal") {
        @Override
        protected Object initialValue() {
            return Maps.newHashMap();
        }
    };

    public static Map<String, Object> getThreadLocal() {
        return threadLocal.get();
    }

    public static <T> T get(String key) {
        Map map = threadLocal.get();
        return (T) map.get(key);
    }

    public static void set(String key, Object value) {
        Map map = threadLocal.get();
        map.put(key, value);
    }

    public static void set(Map<String, Object> kvMap) {
        Map map = threadLocal.get();
        map.putAll(kvMap);
    }

    public static void remove(String key) {
        Map map = threadLocal.get();
        map.remove(key);
    }

    public static void removeAll() {
        Map map = threadLocal.get();
        map.clear();
        threadLocal.remove();
    }

}
```