---
title: RxJava响应流程
date: 2017-07-15 21:03:28
tags:
---

#### 什么是响应式编程？

响应式编程就是与**异步数据流交互**的编程范式

比较通俗的理解的话，性质和富士康里面的流水线差不多，只有当这一个步骤操作完毕之后才会进入到下个步骤。不过这里使用的是异步操作，每一个步骤可以无关其他步骤的操作结果。

<!-- more -->

#### Rxjava原理

rxjava 工作流程就类似于两根水管之间的关系

Observable 就是上游(被观察者) 发送是事件

Observer 就是下游(观察者) 接收事件

它们之间的关系通过subscribe(订阅)产生联系

##### 事件规则

上游可以无限制的发送任意的事件，不过在规则方面有一定限制：

- 上游发送一个onComplete/onError事件之后，下游就不会收到onCompele/onError之后的事件了
- 上游可以不发送onComplete或者onNext
- onCompele和onError唯一且互斥
  - 发送多个onComplete可以正常运行
  - 发送多个onError，收到后面事件后会引起程序崩溃

##### Rxjava2.x相比1.x都有哪些不同

- Subsriber => ObservableEmitter  事件发射器
  - 用来发送事件,可以发出Next、Complete、Error事件
- Subcription => Disposable 订阅处理器
  - 订阅的时候通过onSubscribe方法作为参数传入，可以调用dispose()方法切断订阅关系。
  - 切断订阅关系后，下游不会再收到事件。
- Action/Func => 更改为Java8命名格式
  - Action1更名为Consumer，只关心onNext事件
- 新增Flowable => 专门应对背压问题
  - 背压:生产者速度大于消费者速度带来的问题 ，如android 常见的点击过快造成多次点击两次的效果。

##### 隐藏问题

​	subscribe时Activity退出，回到主线程更新ui，应用崩溃。

​	**解决：**利用onSubscribe方法中传入的Disposable对象在Acitivity退出时切断订阅关系。

​	如果有多个Disposable，可以使用RxJava内置容器CompositeDisposable进行管理。每次得到一个Disposable就add到容器中，退出时，调用CompositeDisposable.clear()即可切断所有订阅关系。

#### 操作符

- map

  > 转换事件，使每一个事件都按照指定的函数进行转换。	
  >
  > 可以将事件类型转换为任意类型。

- flatMap

  > 把一个Observable转换为另一个Observable，把多个操作连贯起来。
  >
  > 将每一个事件单独拿出来，转换为多个事件，然后再将多个事件合并为一个Observable，最后将多个Observable发送出去。
  >
  > 因为faltMap做这些Observable发射的数据做的是合并操作，所以FlatMap并不保证事件的顺序

  **tip:map和flatMap的区别**

  map是在一个item被发射之后，到达map处经过转换变成另一个item，然后继续往下走。

  flatMap是item被发射之后，到达flatMap处经过转换改变为一个Observable，而这个Observable并不会直接被发射出去，而是会立即激活，然后把它发射出的每个item都传入流中，再继续走下去。

  所以区别有以下两个：

  1. 进过Observable的转换，想到与重新开了一个异步的流。
  2. item被分散了，个数发生了变化。

- concatMap

  > 和flatMap作用一样，不过保证事件的发送顺序

- doOnNext

  > 允许在每次输出一个元素之前做一些额外的事情。在onNext之前调用

- zip

  > 通过一个函数将多个Observable发送的事件结合到一起，然后发送这些组合到一起的事件。
  >
  > 组合的顺序是严格按照事件发送顺序来进行的。
  >
  > 最终收到的事件数量取决于事件最少的Observable中的事件数量。
  >
  > 默认两个Observable在同一个线程中，所以造成一个Observable事件发送完毕后，第二个Observable才开始发送。
  >
  > 只要有一个Observable发送了Complete事件，两个Observable中的剩余事件都不会继续发送。

#### 背压

> 当在异步线程中发送的事件数量大于接收处理的事件数量，那么其他未处理的事件就会暂时存储在内存中，数量过多后就会引起内存溢出的问题。

##### 解决方法

控制数量：

​	使用sample()方法，在所有事件中进行取样,选择数量的事件传递到订阅者。

控制速度：

​	每次发送事件的时候设置发送间隔。

Flowable：

​	用Flowable替换Observable,Subscriber替换Observer

#### Flowable

​	采用`响应式拉取`的方式解决上下游流速不均衡的问题。

​	在onSubscribe()的回调中使用request()方法，指定下游的事件处理能力。这样上游就会根据下游的处理能力发送事件。

​	同步代码中，如果没有调用request()方法的话，上游就认为下游没有处理事件的能力，抛出MissingBackpressureException异常。

​	异步代码中，如果没有调用request的话。上游会默认将事件放到一个大小为128的暂存池里面，等下游调用了request时，才会从暂存池里面取出事件，发送给下游。

##### 处理策略

> 在Flowable.create(new FlowableOnSubcribe() , BackpressureStategy.Buffer)时指定

**BackpressureStrategy.Error**

​	出现上下游流速不均衡的情况时，抛出MissingBackpressureException异常。默认事件的缓存大小为128

**BackpressureStrategy.Buffer**
​	缓存任意事件直到下游开始消耗

**BackpressureStrategy.Drop**

​	把超过128之后的事件直接丢弃

**BackpressureStrategy.Latest**

​	只保留最新的事件	

如果不是自己创建的Flowable，可以在之后调用

- onBackpressureBuffer()
- onBackpressureDrop()
- onBackPressureLatest()

指定策略模式

##### 异步情况

​	在异步线程中一开始系统会自动调用request方法指定Io线程中的requested的值。当上游的requested的值减少到一定数量的时候，系统就会再次出发request方法，设置requested的值。

​	当下游每消费96个事件之后便会自动出发内部的request()方法，去设置上游的requested的值。

​	

​	