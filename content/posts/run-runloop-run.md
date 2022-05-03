---
title: "Run, Run Loop, Run!"
date: 2016-09-09T19:37:36+08:00
draft: false
---

> _翻译自 [http://bou.io/RunRunLoopRun.html](http://bou.io/RunRunLoopRun.html)_

尽管作为所有App最重要的组成部分之一，它仍是一个开发者很少被触及讨论的话题：Run Loop。Run Loop之于app的就像是跳动的心脏，他们是使你app运行的真正奥义。

事实上，一个run loop（运行循环）最基本的原则非常简单。在iOS和OS X[^1]，`CFRunLoop`实现被更高层级发送和分发消息API使用的核心机制。

# 所以，Run Loop到底是什么？
<!--more-->

简单的说，一个run loop是一个消息机制，用来异步或者线程间通信。可以被看作是一个等待信件的邮箱，并把他们分发到接受者。

一个run loop做两件事：
* 等待事件发生（比如，消息到达）
* 把消息分发给它的接受者

在别的平台上[^2]，这种机制叫作Message Pump。

Run Loops是从命令行工具中分离出来的交互式应用程序。命令行工具通过带参数启动，执行命令，然后退出。交互式应用等待用户输入，作出响应，然后恢复等待。其实，这种基本的机制可以在久驻内存的进程中看到。运行于服务器的一个run loop就比如`while(1) { select(); }`[^3](#fn:3)。

Run loop的工作就是等待事件发生。这些事件可以是外部事件，由用户或者系统触发（比如网络请求）或者应用内消息（比如线程间通知、异步代码执行、timer计时器。一旦一个事件（或者叫“消息”）到达，run loop会寻找一个相关的监听器（listener）并传递事件。

一个基本的run loop实现起来非常简单。下面是一段伪代码：
```
func postMessage(runloop, message)
{
    runloop.queue.pushBack(message)
    runloop.signal()
}
func run(runloop)
{
    do {
        runloop.wait()
        message = runloop.queue.popFront()
        dispatch(message)
    } while(true)
}
```

通过这个简单的机制，每个线程都可以`run()`它们自己的run loop，并且和其他线程的run loop通过`postMessage()`异步地交互信息。我的同事[Cyril Mottier](http://www.opensource.apple.com/source/CF/CF-855.17/CFRunLoop.c)指出[Android的实现](https://github.com/android/platform_frameworks_base/blob/master/core/java/android/os/Looper.java#L110-L155)并不比这个复杂太多。

# 那 iOS 和 OS X 呢


在苹果的系统中，这是 **`CFRunLoop`** 的工作，以一种[稍微高级的变体](http://www.opensource.apple.com/source/CF/CF-855.17/CFRunLoop.c)[^4]的方式。_从某种程序上来说，你所写的代码都是被`CFRunLoop`调用的_，除了最开始的初始化，或者你自己创建线程。（就我所知，线程为Grand Central Dispatch自动创建，不需要`CFRunLoop`，但可以确定有一个消息系统来允许重用）。

CFRunLoop最重要的特性就是 **CFRunLoopModes**。CFRunLoop与Run Loop源（Run Loop Sources）交互工作。源在一个run loop上注册一个或者多个 __模式（mode）__，run loop会在给定的一个模式下运行。当一个源上的事件到达时，如果源模式符合run loop当前模式，它才会被这个run loop处理。

另外，在应用的代码中，**CFRunLoop是可以被重入的**，无论是你自己的代码还是框架中的代码。因为每个线程只有一个run loop，当一个组件想在一个特定的模式运行时，它只需调用 `CFRunLoopRunInMode()` 即可。此时没有被注册到这个模式的run loop源会被简单的执行停止操作。通常，那个组件最终会把控制权返回给之前的模式。

CFRunLoop定义了一个伪模式（pseudo-mode）叫作“common modes”（`kCFRunLoopCommonModes`），它实际上是一个应用中包含“正常”run loop模式的集合。最开始，主run loop会运行在`kCFRunLoopCommonModes`模式下。

另一方面，UIKit定义了一个叫作`UITrackingRunLoopMode`的特殊run loop模式。“当跟踪控件时”会使用这个模式，比如触摸事件。这很重要，因为它保证了tableview很顺滑的滚动。当主线程的run loop在`UITrackingRunLoopMode`模式下，比如网络请求回调等是不会被分发处理的。没有其他操作进行的方式保证滚动时不会卡顿。（好吧，至少现在那是你的错误观点[^5]。）

# CFRunLoop解迷

如果你已经在iOS或者OS X调试过调用栈，你可以已经注意到，在调用栈的底部，一个全大写的消息以 `CFRUNLOOP_IS_CALLING_OUT` 开头。当CFRunLoop调用应用代码时，就像是做一个关于它的表演。在CFRunLoop.c中一共定义了6种这种的函数：
```objc
static void __CFRUNLOOP_IS_CALLING_OUT_TO_AN_OBSERVER_CALLBACK_FUNCTION__();
static void __CFRUNLOOP_IS_CALLING_OUT_TO_A_BLOCK__();
static void __CFRUNLOOP_IS_SERVICING_THE_MAIN_DISPATCH_QUEUE__();
static void __CFRUNLOOP_IS_CALLING_OUT_TO_A_TIMER_CALLBACK_FUNCTION__();
static void __CFRUNLOOP_IS_CALLING_OUT_TO_A_SOURCE0_PERFORM_FUNCTION__();
static void __CFRUNLOOP_IS_CALLING_OUT_TO_A_SOURCE1_PERFORM_FUNCTION__();
```

正如你所猜测的，这些函数的主要作用就是帮助在调用栈里调试。CFRunLoop确保所有的应用代码都会被这些函数中的某一个调用。

我们一个一个的看。
```objc
static void __CFRUNLOOP_IS_CALLING_OUT_TO_AN_OBSERVER_CALLBACK_FUNCTION__(
    CFRunLoopObserverCallBack func,
    CFRunLoopObserverRef observer,
    CFRunLoopActivity activity,
    void *info);
```

**观察者**有点特殊。你可以通过`CFRunLoopObserver`API观察CFRunLoop的行为，并得到它的活动的通知：当它在处理事件时，当它准备休眠时等等。这在调试时会显得非常有用，你可能在你的应用中并不需要它，但如果你想了解CFRunLoop的特性时你就可以这样做。

> 2014-10-02更新：  
> 事实上，它们在特定情形下会很有用。比如，CoreAnimation从它的观察者处被调用。它是有意义的：通过确定所有的UI代码都已经执行，它会执行起所有的动画。

```objc
static void __CFRUNLOOP_IS_CALLING_OUT_TO_A_BLOCK__(void (^block)(void));
```

Blocks就是`CFRunLoopPerformBlock()`API的中的block，你想在“下一次运行循环”时运行你的代码会用到这个。
```objc
static void __CFRUNLOOP_IS_SERVICING_THE_MAIN_DISPATCH_QUEUE__(void *msg); 
```

这个 **主调度队列(Main Dispatch Queue)** 当然是CFRunLoop在和GCD交互时使用的。显然，至少在主线程中GCD和CFRunLoop会协同工作。尽管CGD能并且也会建立没有CFRunLoop的线程，但当存在一个线程的时候，一个CFRunLoop也就存在了[^6]。
```objc
static void __CFRUNLOOP_IS_CALLING_OUT_TO_A_TIMER_CALLBACK_FUNCTION__(
    CFRunLoopTimerCallBack func,
    CFRunLoopTimerRef timer,
    void *info
);
```

Timer顾名思义定时器。在iOS和OS X上，高级的定时器像NSTimer或者`performSelector:afterDelay:`是使用CFRunLoop定时器实现的。从iOS7和Mavericks开始，Timer对它的触发时间有一个_宽容度_，这个特性也是由CFRunLoop处理的。
```objc
static void __CFRUNLOOP_IS_CALLING_OUT_TO_A_SOURCE0_PERFORM_FUNCTION__(
    void (*perform)(void *),
    void *info
);
```

`CFRunLoopSource` 的“Version 0”和“Version 1”实际上是非常不同的东西，尽管他们有着相同的API。**Version 0源**只是一个简单的应用内的消息机制，并且必须通过应用程序代码来手动处理。在通过`CFRunLoopSourceSignal()`发出一个Version 0源的信号后，CFRunLoop必须通过`CFRunLoopWakeUp()`被唤醒来为这个源处理事件。
```objc
static void __CFRUNLOOP_IS_CALLING_OUT_TO_A_SOURCE1_PERFORM_FUNCTION__(
    void *(*perform)(void *msg, CFIndex size, CFAllocatorRef allocator, void *info),
    mach_msg_header_t *msg, CFIndex size, mach_msg_header_t **reply,
    void (*perform)(void *),
    void *info
);
```

与之相对应，**Version 1源**使用`mach_ports`处理内核事件。这是CFRunLoop的核心。多数时候，当你的应用停留在那里什么也不做，它是被`mach_msg(...,MACH_RCV_MSG,...)`这个调用所阻塞。如果你通过Activity Monitor（中文名：活动监视器）从任何一个app进行取样，你可能会遇到这样的情形：
```objc
2718 CFRunLoopRunSpecific  (in CoreFoundation) + 296  [0x7fff98bb7cb8]
  2718 __CFRunLoopRun  (in CoreFoundation) + 1371  [0x7fff98bb845b]
    2718 __CFRunLoopServiceMachPort  (in CoreFoundation) + 212  [0x7fff98bb8f94]
      2718 mach_msg  (in libsystem_kernel.dylib) + 55  [0x7fff99cf469f]
        2718 mach_msg_trap  (in libsystem_kernel.dylib) + 10  [0x7fff99cf552e]
```

它在CFRunLoop.c的[这个位置](https://github.com/opensource-apple/CF/blob/master/CFRunLoop.c#L2021)。往上几行，可以看到来自苹果工程师从[哈姆雷特的独白](https://en.wikipedia.org/wiki/To_be,_or_not_to_be)中引用的一段相关的话：
```objc
/* In that sleep of death what nightmares may come ... */
```

# 瞧一瞧CFRunLoop.c

无论何时你运行你的程序，CFRunLoop的核心是`__CFRunLoopRun()`这个函数，通过公有API`CFRunLoopRun()`和`CFRunLoopRunInMode(mode, seconds, returnAfterSourceHandled)`调用。

以下原因会使`__CFRunLoopRun()`退出：

* `kCFRunLoopRunTimedOut`：超时的时候，如果指定了一个时间间隔；
* `kCFRunLoopRunFinished`：它变成“空”的时候，比如所有的源都被移除了；
* `kCFRunLoopRunHandledSource`：指定了`returnAfterSourceHandled`标志，在一个事件被分发完成的同时；
* `kCFRunLoopRunStopped`：通过手动调用`CFRunLoopStop()`使其停止时；

在上面的任何一个原因出现之前，run loop都会持续等待并分发事件。下面看一下处理我们上面讨论过的各种事件类型的简单过程是什么样的：

1.  调用“blocks”，（通过`CFRunLoopPerformBlock()`API）
2.  检查Version 0源，如果需要会调用它们的“执行”函数
3.  轮询、内部调度队列和`mach_ports`
4.  如果没有可等待的事件则休眠。内核会在有事件发生的时候唤醒它。事实上代码里比这复杂得多，因为a)一个Win32兼容的代码会添加很多`#ifdef` `#elfi`条件编译选项并且b)代码中间有`goto`。主要的想法是`mach_msg()`可以被配置为等待多个队列和端口。CFRunLoop凭此可以同时等待所有的Timer，GCD调度，手动唤醒或者Version 1源。
5.  唤醒，来看看为什么：
    1.  手动唤醒。只是继续运行这个循环，可以是block或者Version 0源需要服务
    2.  一个或者多个Timer启动。调用它们的函数。
    3.  GCD需要工作。通过一个指定的“4CF”调度队列API来调用
    4.  一个Version 1源被内核发送。查找并服务它
6.  再次调用“blocks”
7.  检查退出条件：Finished, Stopped, TimedOut, HandledSource
8.  从头开始

哟，很简单吧？你可能知道，CoreFoundation是由C实现的，坦率地说看起来并不是很现代。读完我的第一反应就是“哇，这个需要重构”。另一方面，这个代码不仅仅是场景测试，所以我也不指望后面会使用Swift的全部重写。

这是我这些年用的很多的一个代码模式，尤其是在测试里。它“一直运行直到这个条件变成真”，这是任何一种异步单元测试的基础。随着时间的推移，我可能与了很多它的变体，直接使用NSRunLoop或者CFRunLoop，做轮询，使用超时等等。现在我可以写一个它的像样的版本。我们下一篇文章里面再看～

[^1]: 我们需要一个苹果系统家族的名字。"Darwin"不太对，因为它是底层系统的名字。OS？
[^2]: 在此之前，我写了很久的 Win32 代码。
[^3]: 如果你恰好不知道，可以`man 2 select`。
[^4]: CFRunLoop.c有3909行，而Looper.java只有309行。
[^5]: 你可以还记得2011年的 [这篇文章](https://plus.google.com/u/0/+AndrewMunn/posts/VDkV9XaJRGS) 把 iOS 上的滚动性能归功于“提升 UI 线程到一个实时的优先级”。后来被纠正，当然，那只是UITrackingRunLoopMode在完成它的工作。
[^6]: 说实话，我并不是很熟悉GCD的内部实现。我只是猜测它是如何工作的，如果错了请纠正我。

