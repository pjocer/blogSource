---
title: RAC API个人总结(一)
date: 2016-04-20 15:30:31
categories: iOS
tags: framework
---


### 一、常见类

#### RACSignal

- RACEmptySignal ：空信号，用来实现 RACSignal 的 +empty 方法；


- RACReturnSignal ：一元信号，用来实现 RACSignal 的 +return: 方法；


- RACDynamicSignal ：动态信号，使用一个 block - 来实现订阅行为，我们在使用 


- RACSignal 的 +createSignal: 方法时创建的就是该类的实例；


- RACErrorSignal ：错误信号，用来实现 RACSignal 的 +error: 方法；


- RACChannelTerminal ：通道终端，代表 RACChannel 的一个终端，用来实现双向绑定。

<!-- more -->

#### RACDisposable
RACDisposable 用于取消订阅或者清理资源，当信号发送完成或者发送错误的时候，就会自动触发它。

- RACSerialDisposable ：作为 disposable 的容器使用，可以包含一个 disposable 对象，并且允许将这个 disposable 对象通过原子操作交换出来；

- RACKVOTrampoline ：代表一次 KVO 观察，并且可以用来停止观察；

- RACCompoundDisposable ：它可以包含多个 disposable 对象，并且支持手动添加和移除 disposable 对象

- RACScopedDisposable ：当它被 dealloc 的时候调用本身的 -dispose 方法。

#### RACSubject 
信号提供者，自己可以充当信号，又能发送信号。

- RACGroupedSignal ：分组信号，用来实现 RACSignal 的分组功能；

- RACBehaviorSubject ：重演最后值的信号，当被订阅时，会向订阅者发送它最后接收到的值；

- RACReplaySubject ：重演信号，保存发送过的值，当被订阅时，会向订阅者重新发送这些值。

#### RACTuple
RACTuple 元组类,类似NSArray,用来包装值.

#### RACSequence
RAC中的集合类

#### RACCommand 
RAC中用于处理事件的类，可以把事件如何处理,事件中的数据如何传递，包装到这个类中，他可以很方便的监控事件的执行过程。

#### RACMulticastConnection
用于当一个信号，被多次订阅时，为了保证创建信号时，避免多次调用创建信号中的block，造成副作用，可以使用这个类处理。

#### RACScheduler
RAC中的队列，用GCD封装的。

- RACImmediateScheduler ：立即执行调度的任务，这是唯一一个支持同步执行的调度器；

- RACQueueScheduler ：一个抽象的队列调度器，在一个 GCD 串行列队中异步调度所有任务；

- RACTargetQueueScheduler ：继承自 RACQueueScheduler ，在一个以一个任意的 GCD 队列为 target 的串行队列中异步调度所有任务；

- RACSubscriptionScheduler ：一个只用来调度订阅的调度器。

### 二、常见用法

- rac_signalForSelector : 代替代理
- rac_valuesAndChangesForKeyPath: KVO
- rac_signalForControlEvents:监听事件
- rac_addObserverForName 代替通知
- rac_textSignal：监听文本框文字改变
- rac_liftSelector:withSignalsFromArray:Signals:当传入的Signals(信号数组)，每一个signal都至少sendNext过一次，就会去触发第一个selector参数的方法。与combineLatest:方法相同


### 三、常见宏

- RAC(TARGET, [KEYPATH, [NIL_VALUE]])：用于给某个对象的某个属性绑定

- RACObserve(self, name) ：监听某个对象的某个属性,返回的是信号。

- @weakify(Obj)和@strongify(Obj)

- RACTuplePack ：把数据包装成RACTuple（元组类）

- RACTupleUnpack：把RACTuple（元组类）解包成对应的数据

- RACChannelTo 用于双向绑定的一个终端

### 四、常用操作方法


* *容易混淆的方法*

> ## **map与flattenMap** 
>> flattenMap一般用于将某一信号的值映射成另一个信号
>> map更多用于直接映射信号的值，信号本身不变;
>> flatten: 信号的最大订阅数量

> **concat:** 
>> a:concat:b 只有在b信号sendCompelete之后，a信号才会执行


> ## repeat,retry,retry:  
>>repeat,重复执行信号;
>>retry,若信号sendError则重复执行信号;
>>retry: 信号sendError后最多执行n次

> ## combineLatest:/combineLatestWith:/combineLatest:reduce:
>> 绑定一个信号、绑定多个信号、绑定多个信号并聚合成一个值 ————形成的新的信号需要所有信号都sendNext至少一次，才会sendNext
 
> ## merge:/zip:/
>> 合并单个或多个信号，每个信号sendNext时，合并后的信号都会收到,相当于订阅所有的信号;
>> zip: 压缩信号，将每个信号的值包裹成RACTuple，压缩后的信号只有当被压缩的信号都sendNext时，才会sendNext

> try/catch 用法类似c++,捕捉异常，订阅Error
> 
> switchToLatest 一般用于当信号的值是一个信号时，直接订阅信号的值信号。
> 
> 







