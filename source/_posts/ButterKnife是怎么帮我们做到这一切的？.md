---
title: ButterKnife是怎么帮我们做到这一切的？
date: 2017-01-20 22:19:53
tags: Android
---



### **职责划分**

在分析原理之前我们可以根据职责将功能实现分为两个部分

- **编译期**对源代码进行处理，动态生成ViewBinder类。
- **运行期**对这些新生成的类进行调用，注入数据到指定的类中。

### **编译期操作**

> 在开始看源码之前那首先我们要明确java中注解的三种不同的生命周期。
>
> - SOURCE： 源码时保留，大多在起校验作用。如：Override、Deprecated
> - RUNTIME运行时保留，可以通过反射机制读取注解的信息
> - CLASS：编译时保留，在class文件中有效，可以在编译时用来生成辅助代码

<!-- more -->

#### 前提

一般来说依赖注入框架常使用的套路都是运行时通过反射完成注入，不过这样的话对性能影响很大。

而我们今天的ButterKnife则完全避开了这个问题，它使用的是一个名叫编译时解析技术，也就是APT（Anotation Processing Tool）。

这种技术是说在编译器开始工作的时候，会自动查找代码中所有表明生命周期为CLASS的注解，然后找到该注解注册的Processor类，主动调用这个类的process()方法。

我们常见的辅助类XXXActivity_ViewBinding就是在这个时候创建的~

 **代码分析：**

​	我们首先进入ButterKnifeProcessor类中,找到系统主动调用的process()方法

![img](http://ol8oi6yv0.bkt.clouddn.com/static/images/ButterKnifeImage.png)

然后我们进入查找注解的方法findAndParseTaregets(env)方法中。

因为这里会将所有类型的注解都拿出来做判断，所以我们就只取其一找到遍历@BindView注解的这部分代码。

![img](http://ol8oi6yv0.bkt.clouddn.com/static/images/ButterKnifeImage%281%29.png)

接着进入parseBindView()查看解析过程 

在这个方法中会先对注解元素进行类型判断

![img](http://ol8oi6yv0.bkt.clouddn.com/static/images/ButterKnifeImage%282%29.png)


然后再获取注解绑定元素的基本信息，因为此时我们走的时@BindView注解的流程，所以对应获取的也是View的ID。

![img](http://ol8oi6yv0.bkt.clouddn.com/static/images/ButterKnifeImage%284%29.png)

之后我们再进入getOrCreateBindingBuilder()方法中，这个方法返回一个Builder实例。

我们继续深入，点击newBuilder()方法，进入了Builder的实现代码。

这里就是将我们所需要的所有的基本信息获取到，然后再用获取到的信息构建Builder对象的过程。

![img](http://ol8oi6yv0.bkt.clouddn.com/static/images/ButterKnifeImage%285%29.png)

现在让我们回到刚开始的时候 

此时的Map里面就已经储存有了所有的注解元素。

接着就是用Binding中储存的信息生成辅助类了。

不过这里继续点下去的话就到另外的一个框架了，咱是我们就先到这里。

![image(7)](http://ol8oi6yv0.bkt.clouddn.com/static/images/ButterKnifeImage%287%29.png)

### **运行时操作**

​	运行期进行的操作就比较简单了，首先从ButterKnife.bind()方法进入。

​	发现会调用其他多参bind()方法，我们继续点过去。

​	接着我们发现，这里首先是通过反射拿到了我们传入进去的Activity类的Class对象，然后又以刚才拿到的Class对象的类型+'$$ViewBinder'反射出来了一个viewBinder 。接着又调用了该对象的bind()方法。

​	而ViewBinder就是我们在编译时自动生成的代码，调用此ViewBinder的bind()方法后你会发现，其实它也是使用findViewById(id)的方式获取到View控件的。只不过是它在获取到控件之后又注入到注解元素中的，所以这一切其实我们都没有省略，只不过是ButterKnife这个可爱的框架帮我们代劳了。





