---
title: RAC API个人总结(二)
date: 2016-04-24 13:46:04
tags: framework
categories: iOS
---

# RACScheduler和RACSequence

***

## RACScheduler

`RACScheduler`作为调度器，底层只是对GCD的简单的封装，下面是`RACScheduler`的类簇图：

![RACScheduler](https://raw.githubusercontent.com/pjocer/blogSource/master/RAC-API%E4%B8%AA%E4%BA%BA%E6%80%BB%E7%BB%93-%E4%BA%8C/RACScheduler.png)

<!-- more -->

`RACScheduler`的子类：

>* immediateScheduler：仅仅是一个单例类，作为同步调度器；
>* RACQueueScheduler：一个抽象类，在一个串行队列中实现异步调用。
    一般使用它的子类RACTargetQueueScheduler；
>* RACTargetQueueScheduler：继承于RACQueueScheduler，在一个串行队列中实现异步调用。
>* RACSubscriptionScheduler：一个用来调度订阅的调度器。底层生成了_backgroundScheduler，而
_backgroundScheduler是通过RACTargetQueueScheduler生成的，因此RACSubscriptionScheduler
其实也是间接调用了RACTargetQueueScheduler。


可以看到`RACScheduler`还是挺简单的。下面分三部分剖析`RACScheduler`：
>* RACScheduler生成的一般调度器；
>* 	RACScheduler生成的定时器；
>* RACScheduler的递归调度scheduleRecursiveBlock；

`RACScheduler`的生成的一般调度器有哪些？
>* immediateScheduler
>* mainThreadScheduler
>* scheduler

#### immediateScheduler

```
+ (instancetype)immediateScheduler {
    static dispatch_once_t onceToken;
    static RACScheduler *immediateScheduler;
    dispatch_once(&onceToken, ^{
        immediateScheduler = [[RACImmediateScheduler alloc] init];
    });
    return immediateScheduler;
}
```
可以看出immediateScheduler是一个单例类，因为immediateScheduler并没有用到CGD的相关操作，可以认为它是一个同步的调度器。

#### mainThreadScheduler
```
+ (instancetype)mainThreadScheduler {
    static dispatch_once_t onceToken;
    static RACScheduler *mainThreadScheduler;
    dispatch_once(&onceToken, ^{
        mainThreadScheduler = [[RACTargetQueueScheduler alloc] initWithName:@"com.ReactiveCocoa.RACScheduler.mainThreadScheduler" targetQueue:dispatch_get_main_queue()];
    });
    return mainThreadScheduler;
}
```

mainThreadScheduler也是单例类，看一下initWithName:targetQueue:的实现：


```
- (id)initWithName:(NSString *)name targetQueue:(dispatch_queue_t)targetQueue {
    NSCParameterAssert(targetQueue != NULL);
    if (name == nil) {
        name = [NSString stringWithFormat:@"com.ReactiveCocoa.RACTargetQueueScheduler(%s)", dispatch_queue_get_label(targetQueue)];
    }
    dispatch_queue_t queue = dispatch_queue_create(name.UTF8String, DISPATCH_QUEUE_SERIAL);
    if (queue == NULL) return nil;
    dispatch_set_target_queue(queue, targetQueue);
    return [super initWithName:name queue:queue];
}
```

dispatch_set_target_queue一般有两个作用：

* 用来给新建的queue设置优先级和targetQueue的相同；
* 修改用户队列的目标队列；

显然这里是把新建的queue加入dispatch_get_main_queue()中，因此可以理解mainThreadScheduler是任务运行在主队列中的调度器。

#### scheduler

```
+ (instancetype)scheduler {
    return [self schedulerWithPriority:RACSchedulerPriorityDefault];
}
+ (instancetype)schedulerWithPriority:(RACSchedulerPriority)priority {
    return [self schedulerWithPriority:priority name:@"com.ReactiveCocoa.RACScheduler.backgroundScheduler"];
}
+ (instancetype)schedulerWithPriority:(RACSchedulerPriority)priority name:(NSString *)name {
    return [[RACTargetQueueScheduler alloc] initWithName:name targetQueue:dispatch_get_global_queue(priority, 0)];
}
```

显然，scheduler是任务运行在dispatch_get_global_queue(priority, 0)的调度器，是一个串行的异步队列。

#### RACScheduler的生成的定时器

RACScheduler定时的方法主要有两个：

```
- (RACDisposable *)after:(NSDate *)date schedule:(void (^)(void))block {
    RACDisposable *disposable = [[RACDisposable alloc] init];
    dispatch_after([self.class wallTimeWithDate:date], self.queue, ^{
        if (disposable.disposed) return;
        [self performAsCurrentScheduler:block];
    });
    return disposable;
}
- (RACDisposable *)after:(NSDate *)date repeatingEvery:(NSTimeInterval)interval withLeeway:(NSTimeInterval)leeway schedule:(void (^)(void))block {
    uint64_t intervalInNanoSecs = (uint64_t)(interval * NSEC_PER_SEC);
    uint64_t leewayInNanoSecs = (uint64_t)(leeway * NSEC_PER_SEC);
    dispatch_source_t timer = dispatch_source_create(DISPATCH_SOURCE_TYPE_TIMER, 0, 0, self.queue);
    dispatch_source_set_timer(timer, [self.class wallTimeWithDate:date], intervalInNanoSecs, leewayInNanoSecs);
    dispatch_source_set_event_handler(timer, block);
    dispatch_resume(timer); 
    return [RACDisposable disposableWithBlock:^{
        dispatch_source_cancel(timer);
    }];
}
```

after:schedule:直接就是调用dispatch_after实现的，没什么好说的；
after:repeatingEvery:withLeeway:schedule:则是通过dispatch_source_t实现的，也没什么好说的。OK，RACScheduler的定时器就这样了。

#### RACScheduler的递归调度scheduleRecursiveBlock

为什么要说scheduleRecursiveBlock？因为在RACSequence内部的很多操作都是通过scheduleRecursiveBlock来完成的，如果要理解RACSequence，那么理解scheduleRecursiveBlock必不可少。
先说一个例子：

```
    NSMutableArray *numbers = [[NSMutableArray alloc] init];
    for (NSInteger i = 0; i < 3; i++) {
        [numbers addObject:[NSNumber numberWithInteger:i]];
    }
    [numbers.rac_sequence.signal subscribeNext:^(id x) {    //(1)
        NSLog(@"%@",x);           //(2)
    }];
结果打印：
0
1
2
```

RACSequence封装了集合的链式操作，用处是非常大的，下篇会结合几个例子说明RACSequence，下面先看源码：

```
- (RACSignal *)signal {
    return [[self signalWithScheduler:[RACScheduler scheduler]] setNameWithFormat:@"[%@] -signal", self.name];
}
- (RACSignal *)signalWithScheduler:(RACScheduler *)scheduler {
    return [RACSignal createSignal:^(id<RACSubscriber> subscriber) {  //(2)
        __block RACSequence *sequence = self;       
        return [scheduler scheduleRecursiveBlock:^(void (^reschedule)(void)) {//(3)
            if (sequence.head == nil) {
                [subscriber sendCompleted];  //(4)
                return;
            }
            [subscriber sendNext:sequence.head];  //(5)
            sequence = sequence.tail;     //(6)
            reschedule();       //(7)
        }];
    }] ;
}
- (void)scheduleRecursiveBlock:(RACSchedulerRecursiveBlock)recursiveBlock addingToDisposable:(RACCompoundDisposable *)disposable {
        [self schedule:^{          //(8)
            void (^reallyReschedule)(void) = ^{  //(9)
                [self scheduleRecursiveBlock:recursiveBlock addingToDisposable:disposable];
            };
            __block NSLock *lock = [[NSLock alloc] init];
            lock.name = [NSString stringWithFormat:@"%@ %s", self, sel_getName(_cmd)];

            __block NSUInteger rescheduleCount = 0;
            __block BOOL rescheduleImmediately = NO;
                recursiveBlock(^{        //(10)
                    [lock lock];         //(11)
                    BOOL immediate = rescheduleImmediately;
                    if (!immediate) ++rescheduleCount;
                    [lock unlock];
                    if (immediate) reallyReschedule();
                });
            }
            [lock lock];
            NSUInteger synchronousCount = rescheduleCount;
            rescheduleImmediately = YES;
            [lock unlock];

            for (NSUInteger i = 0; i < synchronousCount; i++) {
                reallyReschedule();   //(12)
            }
        }];
    }
}
```

在(1)处订阅时，会执行didSubscribe，即(2)处的block，然后到(8)处，经过上面调度器的分析，我们知道schedule是一个串行异步的调度器，因此会在下一个RunLoop执行schedule的代码块。当执行到(10)处时，执行recursiveBlock，我们知道递归包含三个条件：

* **递归的初始条件；**
* **递归函数；**
* **递归的结束条件；**

recursiveBlock就是封装了递归的结束条件，因此递归的结束条件是调度器递归结束的核心。
RACSequence的内部存储结构就像一个单链表，有两个指针head和tail，head指针指向了当前链表的第一个元素，tail指向head指针下一个元素；根据RACSequence是否还有内容来判断是否还需要递归遍历RACSequence（将数组中的元素一个一个发出去），如果RACSequence还有内容，则继续递归，否则信号发送sendCompleted事件，结束整个遍历的过程。
代码(10)处要注意一下，先执行scheduleRecursiveBlock:传进来的参数recursiveBlock，然后在recursiveBlock中执行reschedule---这里有点绕。
在(12)处，如果递归结束了，synchronousCount=0，因此整个递归也就结束了。
RACScheduler的源码分析结束~~



## RACSequence

```
@interface RACSequence : RACStream <NSCoding, NSCopying, NSFastEnumeration>

@property (nonatomic, strong, readonly) id head;
@property (nonatomic, strong, readonly) RACSequence *tail;
@property (nonatomic, copy, readonly) NSArray *array;
@property (nonatomic, copy, readonly) NSEnumerator *objectEnumerator;
@property (nonatomic, copy, readonly) RACSequence *eagerSequence;
@property (nonatomic, copy, readonly) RACSequence *lazySequence;
@end
```

`RACSequence`是`RACStream`的子类，主要是`ReactiveCocoa`里面的集合类。

先来说说关于`RACSequence`的一些概念。

`RACSequence`有两个很重要的属性就是`head`和`tail`。`head`是一个id，而`tail`又是一个`RACSequence`，这个定义有点递归的意味。


```
    RACSequence *sequence = [RACSequence sequenceWithHeadBlock:^id{
        return @(1);
    } tailBlock:^RACSequence *{
        return @[@2,@3,@4].rac_sequence;
    }];

    NSLog(@"sequence.head = %@ , sequence.tail =  %@",sequence.head ,sequence.tail);

输出:
sequence.head = 1 , sequence.tail =  <RACArraySequence: 0x608000223920>{ name = , array = (
    2,
    3,
    4
) }
```

这段测试代码就道出了head和tail的定义。更加详细的描述见下图：

![RACSequence]()







