---
title: DCL单例
date: 2017-07-13 23:18:40
tags: java


---

#### DCL单例(Double Check Lock)

```Java
public class Singleton {   
    
    /**  
     * 单例对象实例  
     */  
    private volatile static Singleton instance = null;   
    
    public static Singleton getInstance() {   
        if (instance == null) {   
            synchronized (Singleton.class) {   
                if (instance == null) {   
                    instance = new Singleton();   
                }   
            }   
        }   
        return instance;   
    }   
}
```

<!-- more -->

##### 使用场景

- 保证同一变量在多线程内存空间里面值的一致性

##### 使用原因

> 在java中所有的变量都存储在主内存中，在线程工作内存中则只保存了此线程使用到的变量的副本。
>
> 而单一线程工作内存在线程之间是隔离的，其他线程不可见。
>
> 并且线程对变量的所有操作都必须在工作内存中进行，修改后的变量副本要写入主内存中。
>
> 这样就会出现同一个变量在某瞬间，在不同线程工作内存中的值可能相互不一致的情况。

##### 实现原理

> 一个变量声明为volatile，就意味着变量被修改的时候其他所有使用到该变量的线程都能立即看到变化(即称为可见性)。
>
> 具体是在每次使用前都要刷新，以保证别的线程中修改的值已经反应到本线程的工作内存中。
>
> 因此可以保证执行时的一致性。

##### 问题

> JDK1.5之前不能完全避免重排序问题
>
> 解决：将变量设置成final，这样在java5中就能够正确运行了。在java5中，final变量一旦在构造函数中设置完成，其它线程必定会看到在构造函数中设置的值。