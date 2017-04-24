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

![RACSequence](https://github.com/pjocer/blogSource/blob/master/RAC-API%E4%B8%AA%E4%BA%BA%E6%80%BB%E7%BB%93-%E4%BA%8C/RACSequence.png?raw=true)

上述代码里面用到了RACSequence初始化的方法，具体的分析见后面。

objectEnumerator是一个快速枚举器。

```
@interface RACSequenceEnumerator : NSEnumerator
@property (nonatomic, strong) RACSequence *sequence;
@end
```

之所以需要实现这个，是为了更加方便的RACSequence进行遍历。

```
- (id)nextObject {
    id object = nil;

    @synchronized (self) {
        object = self.sequence.head;
        self.sequence = self.sequence.tail;
    }

    return object;
}
```

有了这个NSEnumerator，就可以从RACSequence的head一直遍历到tail。



```
- (NSEnumerator *)objectEnumerator {
    RACSequenceEnumerator *enumerator = [[RACSequenceEnumerator alloc] init];
    enumerator.sequence = self;
    return enumerator;
}
```

回到RACSequence的定义里面的objectEnumerator，这里就是取出内部的RACSequenceEnumerator。

```
- (NSArray *)array {
    NSMutableArray *array = [NSMutableArray array];
    for (id obj in self) {
        [array addObject:obj];
    }   
    return [array copy];
}
```

RACSequence的定义里面还有一个array，这个数组就是返回一个NSArray，这个数组里面装满了RACSequence里面所有的对象。这里之所以能用for-in，是因为实现了NSFastEnumeration协议。至于for-in的效率，完全就看重写NSFastEnumeration协议里面countByEnumeratingWithState: objects: count: 方法里面的执行效率了。

在分析RACSequence的for-in执行效率之前，先回顾一下NSFastEnumerationState的定义，这里的属性在接下来的实现中会被大量使用。

```
typedef struct {
    unsigned long state; //可以被自定义成任何有意义的变量
    id __unsafe_unretained _Nullable * _Nullable itemsPtr;  //返回对象数组的首地址
    unsigned long * _Nullable mutationsPtr;  //指向会随着集合变动而变化的一个值
    unsigned long extra[5]; //可以被自定义成任何有意义的数组
} NSFastEnumerationState;
```

接下来要分析的这个函数的入参，stackbuf是为for-in提供的对象数组，len是该数组的长度。

```
- (NSUInteger)countByEnumeratingWithState:(NSFastEnumerationState *)state objects:(__unsafe_unretained id *)stackbuf count:(NSUInteger)len {
    // 定义完成时候的状态为state = ULONG_MAX
    if (state->state == ULONG_MAX) {
        return 0;
    }

    // 由于我们需要遍历sequence多次，所以这里定义state字段来记录sequence的首地址
    RACSequence *(^getSequence)(void) = ^{
        return (__bridge RACSequence *)(void *)state->state;
    };

    void (^setSequence)(RACSequence *) = ^(RACSequence *sequence) {
        // 释放老的sequence
        CFBridgingRelease((void *)state->state);
        // 保留新的sequence，把sequence的首地址存放入state中
        state->state = (unsigned long)CFBridgingRetain(sequence);
    };

    void (^complete)(void) = ^{
        // 释放sequence，并把state置为完成态
        setSequence(nil);
        state->state = ULONG_MAX;
    };

    // state == 0是第一次调用时候的初始值
    if (state->state == 0) {
        // 在遍历过程中，如果Sequence不再发生变化，那么就让mutationsPtr指向一个定值，指向extra数组的首地址
        state->mutationsPtr = state->extra;
        // 再次刷新state的值
        setSequence(self);
    }

    // 将会把返回的对象放进stackbuf中，因此用itemsPtr指向它
    state->itemsPtr = stackbuf;

    NSUInteger enumeratedCount = 0;
    while (enumeratedCount < len) {
        RACSequence *seq = getSequence();
        // 由于sequence可能是懒加载生成的，所以需要防止在遍历器enumerator遍历到它们的时候被释放了

        __autoreleasing id obj = seq.head;

        // 没有头就结束遍历
        if (obj == nil) {
            complete();
            break;
        }
        // 遍历sequence，每次取出来的head都放入stackbuf数组中。
        stackbuf[enumeratedCount++] = obj;

        // 没有尾就是完成遍历
        if (seq.tail == nil) {
            complete();
            break;
        }

        // 取出tail以后，这次遍历结束的tail，即为下次遍历的head，设置seq.tail为Sequence的head，为下次循环做准备
        setSequence(seq.tail);
    }

    return enumeratedCount;
}
```

整个遍历的过程类似递归的过程，从头到尾依次遍历一遍。

再来研究研究RACSequence的初始化：

```
+ (RACSequence *)sequenceWithHeadBlock:(id (^)(void))headBlock tailBlock:(RACSequence *(^)(void))tailBlock;

+ (RACSequence *)sequenceWithHeadBlock:(id (^)(void))headBlock tailBlock:(RACSequence *(^)(void))tailBlock {
   return [[RACDynamicSequence sequenceWithHeadBlock:headBlock tailBlock:tailBlock] setNameWithFormat:@"+sequenceWithHeadBlock:tailBlock:"];
}
```

初始化RACSequence，会调用RACDynamicSequence。这里有点类比RACSignal的RACDynamicSignal。

再来看看RACDynamicSequence的定义。

```
@interface RACDynamicSequence () {
    id _head;
    RACSequence *_tail;
    id _dependency;
}
@property (nonatomic, strong) id headBlock;
@property (nonatomic, strong) id tailBlock;
@property (nonatomic, assign) BOOL hasDependency;
@property (nonatomic, strong) id (^dependencyBlock)(void);

@end
```

这里需要说明的是此处的headBlock，tailBlock，dependencyBlock的修饰符都是用了strong，而不是copy。这里是一个很奇怪的bug导致的。在[ReactiveCocoa](https://github.com/ReactiveCocoa/ReactiveCocoa/issues/505)中详细记录了用copy关键字会导致内存泄露的bug。具体代码如下：

```
[[[@[@1,@2,@3,@4,@5] rac_sequence] filter:^BOOL(id value) {
    return [value intValue] > 1;
}] array];
```

最终发现这个问题的人把copy改成strong就神奇的修复了这个bug。最终整个ReactiveCocoa库里面就只有这里把block的关键字从copy改成了strong，而不是所有的地方都改成strong。

原作者 [Justin Spahr-Summers](https://github.com/jspahrsummers)大神对这个问题的[最终解释](https://github.com/ReactiveCocoa/ReactiveCocoa/pull/506)是：
>Maybe there's just something weird with how we override dealloc, set the blocks from a class method, cast them, or something else.

所以日常我们写block的时候，没有特殊情况，依旧需要继续用copy进行修饰。

```
+ (RACSequence *)sequenceWithHeadBlock:(id (^)(void))headBlock tailBlock:(RACSequence *(^)(void))tailBlock {
   NSCParameterAssert(headBlock != nil);

   RACDynamicSequence *seq = [[RACDynamicSequence alloc] init];
   seq.headBlock = [headBlock copy];
   seq.tailBlock = [tailBlock copy];
   seq.hasDependency = NO;
   return seq;
}
```

hasDependency这个变量是代表是否有dependencyBlock。这个函数里面就只把headBlock和tailBlock保存起来了。

```
+ (RACSequence *)sequenceWithLazyDependency:(id (^)(void))dependencyBlock headBlock:(id (^)(id dependency))headBlock tailBlock:(RACSequence *(^)(id dependency))tailBlock {
    NSCParameterAssert(dependencyBlock != nil);
    NSCParameterAssert(headBlock != nil);

    RACDynamicSequence *seq = [[RACDynamicSequence alloc] init];
    seq.headBlock = [headBlock copy];
    seq.tailBlock = [tailBlock copy];
    seq.dependencyBlock = [dependencyBlock copy];
    seq.hasDependency = YES;
    return seq;
}
```

另外一个类方法sequenceWithLazyDependency: headBlock: tailBlock:是带有dependencyBlock的，这个方法里面会保存headBlock，tailBlock，dependencyBlock这3个block。

从RACSequence这两个唯一的初始化方法之间就引出了RACSequence两大核心问题之一，积极运算 和 惰性求值。

### 积极运算 和 惰性求值

在RACSequence的定义中还有两个RACSequence —— eagerSequence 和 lazySequence。这两个RACSequence就是分别对应着积极运算的RACSequence和惰性求值的RACSequence。

关于这两个概念最最新形象的比喻还是臧老师博客里面的这篇文章[聊一聊iOS开发中的惰性计算](http://williamzang.com/blog/2016/11/07/liao-yi-liao-ioskai-fa-zhong-de-duo-xing-ji-suan/)里面写的一段笑话。引入如下：
>有一只小白兔，跑到蔬菜店里问老板：“老板，有100个胡萝卜吗？”。老板说：“没有那么多啊。”，小白兔失望的说道：“哎，连100个胡萝卜都没有。。。”。第二天小白兔又来到蔬菜店问老板：“今天有100个胡萝卜了吧？”，老板尴尬的说：“今天还是缺点，明天就能好了。”，小白兔又很失望的走了。第三天小白兔刚一推门，老板就高兴的说道：“有了有了，从前天就进货的100个胡萝卜到货了。”，小白兔说：“太好了，我要买2根！”。。。

如果日常我们遇到了这种问题，就很浪费内存空间了。比如在内存里面开了一个100W大小的数组，结果实际只使用到100个数值。这个时候就需要用到惰性运算了。

在RACSequence里面这两种方式都支持，我们来看看底层源码是如何实现的。

先来看看平时我们很熟悉的情况——积极运算。

![EAGER BEAVER]()

在RACSequence中积极运算的代表是RACSequence的一个子类RACArraySequence的子类——RACEagerSequence。它的积极运算表现在其bind函数上。




