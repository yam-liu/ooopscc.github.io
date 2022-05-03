---
title: "replay, replayLast 和 replayLazily 的区别"
date: 2016-07-03T19:41:26+08:00
draft: false
---


> _翻译自 [Patrick Bacon](https://spin.atomicobject.com/author/bacon/)的[Comparing replay, replayLast, and replayLazily](https://spin.atomicobject.com/2014/06/29/replay-replaylast-replaylazily/)_

一位同事最近问我关于 ReactiveCocoa 中`replay`,`replayLast`和`replayLazily`的区别，才发现我只是对它们有一个模糊的理解，并不能把它们的区别解释的很清楚，所以我认为我应该深入了解一下。

你会发现，如果对`RACReplaySubject`和`RACMulticastConnection`没有一个很好的理解的话，只看头文件中的注释还是很难看懂，所以我试图在不涉及的底层概念的情况下解释一下这几个replay方法。
<!--more-->

### 订阅一个信号

对于一个“正常的”`RACSignal`，每个订阅都会使signal的`didSubscribe` block[1](https://ooopscc.github.io/Compare-replay-replayLast-and-replayLazily/#fn:1)再执行一次，并且subscriber只会接收到在订阅**_之后_**发送的值。可能看个例子会更好理解一点。
```objc
__block int num = 0;
RACSignal *signal = [RACSignal createSignal:^RACDisposable *(id  subscriber) {
    num++;
    NSLog(@"Increment num to: %i", num);
    [subscriber sendNext:@(num)];
    return nil;
}];
 
NSLog(@"Start subscriptions");
 
// Subscriber 1 (S1)
[signal subscribeNext:^(id x) {
    NSLog(@"S1: %@", x);
}];
 
// Subscriber 2 (S2)
[signal subscribeNext:^(id x) {
    NSLog(@"S2: %@", x);
}];
 
// Subscriber 3 (S3)
[signal subscribeNext:^(id x) {
    NSLog(@"S3: %@", x);
}];
```

输出为：
```
Start subscriptions
Increment num to: 1
S1: 1
Increment num to: 2
S2: 2
Increment num to: 3
S3: 3
```
[![](https://spin.atomicobject.com/wp-content/uploads/Spin_MarbleChart-05.png)](https://spin.atomicobject.com/wp-content/uploads/Spin_MarbleChart-05.png)
正如你所见，每次订阅都导致`num`的值增加了。而且`num`的值是直到订阅发生了才开始计算的。这样，一个正常的RACSignal可以说是lazy的，因为只有有了subscriber它才会做事。

第二个例子将会展示每个subscriber只接收到 **发送于订阅之后** 的值。
```objc
RACSubject *letters = [RACSubject subject];
RACSignal *signal = letters;
 
NSLog(@"Subscribe S1");
[signal subscribeNext:^(id x) {
    NSLog(@"S1: %@", x);
}];
 
NSLog(@"Send A");
[letters sendNext:@"A"];
NSLog(@"Send B");
[letters sendNext:@"B"];
 
NSLog(@"Subscribe S2");
[signal subscribeNext:^(id x) {
    NSLog(@"S2: %@", x);
}];
 
NSLog(@"Send C");
[letters sendNext:@"C"];
NSLog(@"Send D");
[letters sendNext:@"D"];
 
NSLog(@"Subscribe S3");
[signal subscribeNext:^(id x) {
    NSLog(@"S3: %@", x);
}];
```

输出为：
```
Subscribe S1
 
Send A
S1: A
 
Send B
S1: B
 
Subscribe S2
 
Send C
S1: C
S2: C
 
Send D
S1: D
S2: D
 
Subscribe S
```

[![](https://spin.atomicobject.com/wp-content/uploads/Spin_MarbleChart-02.png)](https://spin.atomicobject.com/wp-content/uploads/Spin_MarbleChart-02.png)

在大多数情况下，这正如预期。但也有一些情况，你不想订阅的代码（subscription code）被重复执行，比如一个订阅是做一个网络请求，当数据回来时有多个监听者需要使用进行更新操作；或者可能是你想得到此次订阅之前的发送过的值。这正是`-replay`，`-replayLast`和`-replayLazily`的使用场景。

### [](https://ooopscc.github.io/Compare-replay-replayLast-and-replayLazily/#订阅一个-replay信号 "订阅一个-replay信号")订阅一个`-replay`信号

`replay`便捷方法会返回一个新的signal，当订阅新的signal时，会直接把在源signal上所有发送的值都发送给subscriber，同时不会重新执行源signal的`didSubscribe` block。当然，这个subscriber也会收到所有将来的值，就像正常的signal一样。

第一个例子会展示在有新的订阅的时候不重新执行`didSubscribe` block：
```objc
__block int num = 0;
RACSignal *signal = [[RACSignal createSignal:^RACDisposable *(id  subscriber) {
    num++;
    NSLog(@"Increment num to: %i", num);
    [subscriber sendNext:@(num)];
    return nil;
}] replay];
 
NSLog(@"Start subscriptions");
 
// Subscriber 1 (S1)
[signal subscribeNext:^(id x) {
    NSLog(@"S1: %@", x);
}];
 
// Subscriber 2 (S2)
[signal subscribeNext:^(id x) {
    NSLog(@"S2: %@", x);
}];
 
// Subscriber 3 (S3)
[signal subscribeNext:^(id x) {
    NSLog(@"S3: %@", x);
}];
```

结果为：
```objc
Increment num to: 1
Start subscriptions
S1: 1
S2: 1
S3: 1
```
[![](https://spin.atomicobject.com/wp-content/uploads/Spin_MarbleChart-06.png)](https://spin.atomicobject.com/wp-content/uploads/Spin_MarbleChart-06.png)

这一次`num`的值在任何订阅产生之前就已经增加了。并且只增加了一次，即这个signal不管有多少subscriber，创建signal的`didSubscribe` block只会执行一次。

第二个例子展示新的subscriber会得到此订阅在这个signal上发送过的所有值。
```objc
RACSubject *letters = [RACSubject subject];
RACSignal *signal = [letters replay];
 
NSLog(@"Subscribe S1");
[signal subscribeNext:^(id x) {
    NSLog(@"S1: %@", x);
}];
 
NSLog(@"Send A");
[letters sendNext:@"A"];
NSLog(@"Send B");
[letters sendNext:@"B"];
 
NSLog(@"Subscribe S2");
[signal subscribeNext:^(id x) {
    NSLog(@"S2: %@", x);
}];
 
NSLog(@"Send C");
[letters sendNext:@"C"];
NSLog(@"Send D");
[letters sendNext:@"D"];
 
NSLog(@"Subscribe S3");
[signal subscribeNext:^(id x) {
    NSLog(@"S3: %@", x);
}];
```

结果为：
```
Subscribe S1
 
Send A
S1: A
 
Send B
S1: B
 
Subscribe S2
S2: A
S2: B
 
Send C
S1: C
S2: C
 
Send D
S1: D
S2: D
 
Subscribe S3
S3: A
S3: B
S3: C
S3: D
```
[![](https://spin.atomicobject.com/wp-content/uploads/Spin_MarbleChart-01.jpg)](https://spin.atomicobject.com/wp-content/uploads/Spin_MarbleChart-01.jpg)

这里尽管S3是在所有值发送之后才订阅的，但它仍旧能获得之前的所有值。

### 订阅一个`-replayLast`信号

`-replayLast`便捷方法返回一个新的signal，当有subscirber订阅这个新signal的时候，它会直接把源signal最近的一个值发送给subscriber，而不需要重新执行signal的`didSubscribe` block。同正常的signal一样，这个subscriber会接收到在此之后的值。

第一个例子中，使用`-replayLast`和`-replay`没有任何区别，所以就不再列一遍了。

第二例子展示最新的值是如何提供给subscriber的。
```objc
RACSubject *letters = [RACSubject subject];
RACSignal *signal = [letters replayLast];
 
NSLog(@"Subscribe S1");
[signal subscribeNext:^(id x) {
    NSLog(@"S1: %@", x);
}];
 
NSLog(@"Send A");
[letters sendNext:@"A"];
NSLog(@"Send B");
[letters sendNext:@"B"];
 
NSLog(@"Subscribe S2");
[signal subscribeNext:^(id x) {
    NSLog(@"S2: %@", x);
}];
 
NSLog(@"Send C");
[letters sendNext:@"C"];
NSLog(@"Send D");
[letters sendNext:@"D"];
 
NSLog(@"Subscribe S3");
[signal subscribeNext:^(id x) {
    NSLog(@"S3: %@", x);
}];
```

输出为：
```
Subscribe S1
 
Send A
S1: A
 
Send B
S1: B
 
Subscribe S2
S2: B
 
Send C
S1: C
S2: C
 
Send D
S1: D
S2: D
 
Subscribe S3
S3: D
```
[![](https://spin.atomicobject.com/wp-content/uploads/Spin_MarbleChart-03.png)](https://spin.atomicobject.com/wp-content/uploads/Spin_MarbleChart-03.png)

### 订阅一个`-replayLazily`信号

`-replayLazily`便捷方法会返回一个新的signal，当被订阅时，会直接发送所有源signal上发送过的值，不重新执行源signal的`didSubscribe` block。`-replayLazily`和`-replay`的不同点在于前者直到有subscriber订阅了新的signal时，它才会去订阅源signal。正好是与`-repaly`和`-replayLast`相反的行为，它俩是一旦被调用会直接订阅源signal。

第一个例子会展示区别。注意“Increment num to: 1”在第一个订阅之后才出现。而`-repaly`和`-replayLazily`时订阅之前就已经出现。
```objc
__block int num = 0;
RACSignal *signal = [[RACSignal createSignal:^RACDisposable *(id  subscriber) {
    num++;
    NSLog(@"Increment num to: %i", num);
    [subscriber sendNext:@(num)];
    return nil;
}] replayLazily];
 
NSLog(@"Start subscriptions");
 
// Subscriber 1 (S1)
[signal subscribeNext:^(id x) {
    NSLog(@"S1: %@", x);
}];
 
// Subscriber 2 (S2)
[signal subscribeNext:^(id x) {
    NSLog(@"S2: %@", x);
}];
 
// Subscriber 3 (S3)
[signal subscribeNext:^(id x) {
    NSLog(@"S3: %@", x);
}];
```
输出为：
```
Start subscriptions
Increment num to: 1
S1: 1
S2: 1
S3: 1
```
[![](https://spin.atomicobject.com/wp-content/uploads/Spin_MarbleChart-07.png)](https://spin.atomicobject.com/wp-content/uploads/Spin_MarbleChart-07.png)
而第二个例子将展示所有发送过的值会发送给任何新的subscriber，就像`-replay`。
```objc
RACSubject *letters = [RACSubject subject];
RACSignal *signal = [letters replayLazily];
 
NSLog(@"Subscribe S1");
[signal subscribeNext:^(id x) {
    NSLog(@"S1: %@", x);
}];
 
NSLog(@"Send A");
[letters sendNext:@"A"];
NSLog(@"Send B");
[letters sendNext:@"B"];
 
NSLog(@"Subscribe S2");
[signal subscribeNext:^(id x) {
    NSLog(@"S2: %@", x);
}];
 
NSLog(@"Send C");
[letters sendNext:@"C"];
NSLog(@"Send D");
[letters sendNext:@"D"];
 
NSLog(@"Subscribe S3");
[signal subscribeNext:^(id x) {
    NSLog(@"S3: %@", x);
}];
```

结果为：
```
Subscribe S1
 
Send A
S1: A
 
Send B
S1: B
 
Subscribe S2
S2: A
S2: B
 
Send C
S1: C
S2: C
 
Send D
S1: D
S2: D
 
Subscribe S3
S3: A
S3: B
S3: C
S3: D
```
[![](https://spin.atomicobject.com/wp-content/uploads/Spin_MarbleChart-04.png)](https://spin.atomicobject.com/wp-content/uploads/Spin_MarbleChart-04.png)

### 总结

ReactiveCocoa提供了三个便捷方法可以使多个subscriber订阅同一个signal而不重新执行源signal的`didSubscribe` block，同时将全部或者最新的发送过的值给新的subscriber。`-replay`和`-replayLast`都会使信号变为热信号，会提供所有发送过的值（`-replay`）或者最新的发送过的值（`-replayLast`）给订阅者。而`-replayLazily`返回一个冷信号，会提供所有发送过的值给订阅者。

---

1.  1.+ (RACSignal _)createSignal:(RACDisposable_ ( ^ ) ( id subscriber ))didSubscribe [↩](https://ooopscc.github.io/Compare-replay-replayLast-and-replayLazily/#fnref:1)

