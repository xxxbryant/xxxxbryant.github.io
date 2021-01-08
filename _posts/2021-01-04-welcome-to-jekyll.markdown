---
layout: post
title: "GCD&NSOperation!"
date: 2021-01-04 17:19:50 +0800
categories: jekyll ios
---

# 多线程编程

在 iOS 开发中，我们常见的几种多线程编程方式

- pthread
- NSThread
- GCD
- NSOperation

这篇博客主要讨论一下 GCD 与 NSOperation。

<!-- more -->

## **常见的面试题：说一说你对串行与并行，同步与异步的理解 ？**

- **串行与并行：**

串行与并行是针对队列而言，而不是线程，队列只负责任务的调度，线程来执行任务。所以，串行队列会对应一个线程，线程会按照先进先出的原则依次执行队列中的任务；并行队列中的任务会被多个线程来同时执行。在 GCD 中由 GCD 中枢来做调度派发线程去执行任务，给串行队列派发一个线程，给并行队列派发多个线程。

- **同步与异步：**

同步与异步的本质：是否会阻塞当前线程。同步会阻塞当前线程；异步不会阻塞当前线程，会开启一个新的线程去执行代码。在代码块中，执行到 sync 时，当前代码所在的线程被阻塞，等待 sync 任务执行完毕，而遇到 async 时，当前代码所在线程不会被阻塞，任务被放到子线程中执行。

# GCD

GCD 的原理，GCD 的底层有一个线程池，线程池中点线程被 GCD 这个中枢来调度派发给队列去执行任务。所以线程池中的线程是可以被重用的，只有当一段时间后，这个线程没有被调用，那么这个线程就会被销毁。开多少条线程是由底层线程池决定的（线程建议控制再 3~5 条），池是系统自动来维护，不需要我们程序员来维护。

> 串行同步：任务出队列后，被被 GCD 中枢从线程池中派发一个线程执行，执行完以后，线程回到线程池。这样队列中的任务反复调度，因为是同步的，所以当我们用 currentThread 打印的时候，就是同一条线程
> 串行异步：任务出队列后，被被 GCD 中枢从线程池中派发一个线程执行，执行完以后，线程回到线程池。虽然是异步，可以开启多个线程，但由于是串行队列，一个任务被执行完才能执行下一个任务，所以依然是 one by one 的执行顺序
> 并行同步：有多个任务在并行队列中可以，但此时只有一个线程，所以中枢所派发的任务只能被这一个线程执行，故而也是 one by one。
> 并行异步：多个任务，多个线程。GCD 中枢从线程池中派发线程去同时执行多个任务，完成一个任务，线程回到线程池等待被中枢派发去执行下一个任务。

- **以上几点得出，只有并行队列异步执行，才会开启新的线程，并发执行任务。**

GCD 中的几个 API

## Group

创建一个组，将不同的队列放到其中，队列中的任务并发执行。当所有任务执行完毕以后，使用 notify 来通知执行完毕，然后处理。

> 队列可以是多个队列，如果是一个串行队列，那么使用组就没有意义了，目的是为了并发执行，但是只有一个串行队列就没有意义了。

`dispatch_group_enter(group)、dispatch_group_leave(group)`
enter 与 leave 必须同时存在，跟信号量差不多，可以理解为引用计数，当调用 enter 时计数加 1，调用 leave 时计数减 1，当计数为 0 时会调用`dispatch_group_notify`并且`dispatch_group_wait`会停止等待； 所以**处理异步任务的同步非常适合使用 enter 与 leave**

> 例如一个页面有多个网络请求，需要这些请求全部完成以后再去刷新 UI，那么此时使用 enter 与 leave 来完成这些异步任务的同步非常适合。
> 如果使用`dispatch_group_async`那么，其中的异步网络请求任务会立刻返回结果，直接进入 notify 与 wait。 无法判断何时请求全部完成，但如果是同步任务，那么效果则一样。

示例代码

<pre>- (void)group {
    dispatch_queue_t dispatchQueue = dispatch_queue_create("ted.queue.next1", DISPATCH_QUEUE_CONCURRENT);
    dispatch_queue_t globalQueue = dispatch_get_global_queue(0, 0);
    dispatch_group_t dispatchGroup = dispatch_group_create();
    dispatch_group_async(dispatchGroup, dispatchQueue, ^(){
        sleep(2);
        NSLog(@"任务一完成");
    });
    dispatch_group_async(dispatchGroup, dispatchQueue, ^(){
        sleep(3);
        NSLog(@"任务二完成");
    });   
    dispatch_group_async(dispatchGroup, globalQueue, ^{     
        sleep(5);
        NSLog(@"任务三完成");
    });  
    dispatch_group_notify(dispatchGroup, dispatch_get_main_queue(), ^(){
        NSLog(@"notify：任务都完成了");
    });
    dispatch_async(dispatch_get_global_queue(0, 0), ^{   
        dispatch_group_wait(disgroup, dispatch_time(DISPATCH_TIME_NOW, 4 * NSEC_PER_SEC));
        NSLog(@"dispatch_group_wait 结束");
    });
}
输出结果为一  二  结束  三  都完成。</pre>

<pre>- (void)group {
    dispatch_group_t group = dispatch_group_create();
    dispatch_group_enter(group);
    dispatch_async(dispatch_get_global_queue(0, 0), ^{       
        sleep(2);
        NSLog(@"任务一完成");
        dispatch_group_leave(group);
    });
    dispatch_group_enter(group);
    dispatch_async(dispatch_get_global_queue(0, 0), ^{
        sleep(8);
        NSLog(@"任务二完成");
        dispatch_group_leave(group);
    });
    dispatch_group_notify(group, dispatch_get_global_queue(0, 0), ^{
        NSLog(@"任务完成");
    });
} 
以上模拟两个网络请求，会等待两个请求都完成 然后执行notify
</pre>

## dispatch_once 创建单例

```
+ (Manager *)sharedInstance {
	static Person *p = nil;
	static dispatch_once_t once;
	dispatch_once($once, ^{
		p = [[Person alloc] init];
	});
	return p;
}
```

## barrier

一组异步并行任务，使用`dispatch_async_barrier` 做插入，该任务会等待前面的任务执行完成以后就去执行，然后再执行后面的任务。async 的区别与 sync 的区别就在于是否阻塞当前线程

<pre>- (void)barrier {
	 dispatch_queue_t concurrentQueue = dispatch_queue_create("my.concurrent.queue", DISPATCH_QUEUE_CONCURRENT);
    dispatch_async(concurrentQueue, ^(){
        NSLog(@"任务1");
    });
    dispatch_async(concurrentQueue, ^(){
        NSLog(@"任务 2");
    });
    dispatch_barrier_async(concurrentQueue, ^(){
        NSLog(@"任务barrier"); 
    });
    dispatch_async(concurrentQueue, ^(){
        NSLog(@"任务3");
    });
    dispatch_async(concurrentQueue, ^(){
        NSLog(@"任务4");
    });
}</pre>

上面输出结果 任务 1，2 一定在 barrier 之前，任务 3，4 在 barrier 之后。

那如果使用`dispatch_barrier_sync`？

<pre>- (void)barrier {
	 dispatch_queue_t concurrentQueue = dispatch_queue_create("my.concurrent.queue", DISPATCH_QUEUE_CONCURRENT);
    dispatch_async(concurrentQueue, ^(){
        NSLog(@"任务1");
    });
    dispatch_async(concurrentQueue, ^(){
        NSLog(@"任务 2");
    });
    dispatch_barrier_sync(concurrentQueue, ^(){
    	 sleep(10);
        NSLog(@"任务barrier"); 
    });
    NSLog(@"222");
    dispatch_async(concurrentQueue, ^(){
        NSLog(@"任务3");
    });
    NSLog(@"111");
    dispatch_async(concurrentQueue, ^(){
        NSLog(@"任务4");
    });
}</pre>

任务 1，2 一定在 barrier 之前，任务 3，4 在 barrier 之后。 同 async 一样。只是 222 与 111 的结果不一样，如果是 async，那么 111，222 会先于 barrier 执行。如果是 sync 那么 111，222 会迟于 barrier 执行。

`dispatch_barrier_sync`和`dispatch_barrier_async`的不共同点：
在将任务插入到 queue 的时候，dispatch_barrier_sync 需要等待自己的任务结束之后才会继续程序，然后插入被写在它后面的任务，接着执行后面的任务。而 dispatch_barrier_async 将自己的任务插入到 queue 之后，不会等待自己的任务结束，它会继续把后面的任务插入到 queue。async 的区别与 sync 的区别就在于是否阻塞当前线程。

## Apply

多次执行一组任务，这些任务的区别差别很小，只有 index 的区分，不必要创建多个 async 的任务。

<pre>NSArray *array = @[@"1", @"2", @"3", @"4", @"5", @"6"];
    dispatch_queue_t queue = dispatch_get_global_queue(0, 0);
    dispatch_async(queue, ^{
        dispatch_apply([array count], queue, ^(size_t index) {
            NSLog(@"%zu : %@", index, [array objectAtIndex:index]);
        });
        dispatch_async(dispatch_get_main_queue(), ^{
            NSLog(@"done");
        });
    });</pre>

上述代码，apply 中的任务会阻塞当前线程，等待 apply 执行完才会执行 done。
**目前对此理解不够**

## 信号量 semaphore

就是一种可用来控制访问资源的数量的标识，设定了一个信号量，在线程访问之前，加上信号量的处理，则可告知系统按照我们指定的信号量数量来执行多个线程。

```
//crate的value表示，最多几个资源可访问
dispatch_semaphore_t semaphore = dispatch_semaphore_create(2);
dispatch_queue_t quene = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);

//任务1
dispatch_async(quene, ^{
	dispatch_semaphore_wait(semaphore, DISPATCH_TIME_FOREVER);// 对信号量进行判读，如果>=1那么可以执行然后进行减一操作，如果信号量<1，那么不执行，直到信号量>=1
	NSLog(@"run task 1");
	sleep(1);
	NSLog(@"complete task 1");
	dispatch_semaphore_signal(semaphore);    信号量进行加一操作
});

//任务2
dispatch_async(quene, ^{
	dispatch_semaphore_wait(semaphore, DISPATCH_TIME_FOREVER);
	NSLog(@"run task 2");
	sleep(1);
	NSLog(@"complete task 2");
	dispatch_semaphore_signal(semaphore);
});

//任务3
dispatch_async(quene, ^{
	dispatch_semaphore_wait(semaphore, DISPATCH_TIME_FOREVER);
	NSLog(@"run task 3");
	sleep(1);
	NSLog(@"complete task 3");
	dispatch_semaphore_signal(semaphore);
});
```

利用信号量的知识，可以做到，将一个异步的任务，做到同步处理。
例如在一个页面中，有多个网络请求，需要多个网络请求同时请求完毕以后，再去执行 UI 绘制，但是无法知晓这几个异步请求合适完成，故而可以使用信号量，将每个请求都封装成一个同步任务。

```
dispatch_semaphore_t semaphore = dispatch_semaphore_create(0);
networkRequest : successBlock{
dispatch_semaphore_signal(semaphore);
}) failureBlock{
	dispatch_semaphore_signal(semaphore);
}
dispatch_semaphore_wait(semaphore, DISPATCH_TIME_FOREVER);  // 只有当走完成功或者失败以后才会执行后面的代码。
```

然后将多个任务请求都用信号量处理，然后将其放到组队列中，当全部完成时，notify 函数便可以执行。

# NSOperation

NSOperation 可以用于创建一个任务，所创建的任务可以被取消。
NSOperation 是一个基类，有两种子类 NSBlockOperation，NSInvocationOperation 一种是 Block 的形式一种是 target，action 的形式。
NSOperation 创建的任务会被默认的添加到主线程当中去。然后 start 这个任务。
NSOperation 有三种属性 isExecuted、isFinished 和 isCancelled，可以通过 KVO 进行属性的观察。
NSOperationQueue 可以设置最大并发数，如果最大并发数设置为 1，那么就被放到主线程中执行，如果大于一那么就会开启子线程。
如果手动创建队列，那么[queue addOperation:operation]会自动帮助 start 这个任务。
任务之间可以添加依赖，如果互相依赖那么就会造成死锁

#NSOperation 与 GCD 的区别与使用

> GCD 更加轻量级。是 C 语言的 API。而 NSOperation 是 OC 对象，是对 GCD 的封装，所以可以通过 KVO 机制监听一个 NSOperation 对象的状态，监听是否完成或者取消。
> NSOperation 可以随时取消已经执行的任务，而 GCD 可以取消或者挂起某个队列
> NSOperation 可以设置依赖关系，让一个 operation 依赖于另一个 operation
> NSOperation 可以设置任务的优先级，而 GCD 可以设置队列的优先级
> 自定义一个 NSOperation 类。添加一些属性与方法，可以有更高的自由度
