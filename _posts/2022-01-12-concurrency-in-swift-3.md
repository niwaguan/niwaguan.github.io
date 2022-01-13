---
layout: post
title: 【译】Swift并发编程三
description: 这里是`Swift并发编程系列`第三篇。主要包含`Operation`相关内容。
category:
  - iOS
tags:
  - Swift
  - Operation
  - Thread
---
> 原文地址：https://ali-akhtar.medium.com/concurrency-in-swift-operations-and-operation-queue-part-3-a108fbe27d61

![并发于并行的区别](http://images-for-blog.oss-cn-beijing.aliyuncs.com/2022/01/13/0dawoyzkxm3red8.png)

这里是`Swift并发编程系列`第三篇（共四篇 [一]({{ site.url }}/2021/12/30/concurrency-in-swift-1/)、[二]({{ site.url }}/2021/12/30/concurrency-in-swift-2/)）。包含以下内容：
1. Operations的状态
2. 不同种类的Operations
3. 如何取消Operations
4. 什么是Operation Queue
5. 如何使用Operation Queue管理Operations
6. Operations的依赖管理
7. Operations和GCD的区别


## Operations

`Operations`是以面向对象的方式封装任务，便于异步执行。它可以配合`Operation Queue`使用，也可以单独使用。

`Operation`类是抽象基类，你必须使用其子类来完成任务。它定义了一些你在基类中必须实现的要求。另外，`Foundation`框架提供了两个实现，你可以直接使用。

### Operations 的状态变更

一个`Operation`可以理解为一个`状态机`，也代表了其生命周期：
1. `instantiated`，表示刚创建，创建完成后会转入`isReady`状态。
2. 在执行完`start`方法后，将会转到`isExecuting`状态
3. 在任务完成后，转到`isFinished`状态
4. 在任务完成之前，`cancel`被调用，将会转到`isCancelled`状态

### BlockOperation

这是系统提供的用于并发执行一个或多个`Block`的类。由于支持执行多个`Block`，它会存在类似`group`的特性，只有当`BlockOperation`的所有`block`都执行完后，这个`Operation`才算执行完。在`BlockOperation`中支持高级特性，如：依赖、KVO、通知、取消。

如下图，我们期望其异步执行。但坏消息是，它将阻塞主线程直到任务完成，因为我们在主线程调用了`operation.start()`
![](http://images-for-blog.oss-cn-beijing.aliyuncs.com/2022/01/13/16419627787521.png)

需要注意的是，若`BlockOperation`有多个待执行的任务，他们会并发执行，但依然会阻塞调用线程。如下图：
![](http://images-for-blog.oss-cn-beijing.aliyuncs.com/2022/01/13/16419629870711.png)

如果我们在其他线程调用`start()`方法，它也会将其阻塞：
![](http://images-for-blog.oss-cn-beijing.aliyuncs.com/2022/01/13/16419632928864.png)

我们可以添加`Operation`完成后执行的任务，他将在所有任务并发执行后执行：
![](http://images-for-blog.oss-cn-beijing.aliyuncs.com/2022/01/13/16419634977469.png)

### NSInvocationOperation

在`Objective-C`中的`NSInvocationOperation`在`Swift`中不能使用。因为其依赖的`NSInvocation`及相关`api`在`Swift`中不可用。
![-w922](http://images-for-blog.oss-cn-beijing.aliyuncs.com/2022/01/13/16419638386237.jpg)

### 自定义的 Operations

自定义`Operations`可以实现状态的完全控制。如下，我们自定义的`Operation`覆写了`main`方法，从而实现`非并发`的`Operation`。它也会阻塞其调用线程。
![](http://images-for-blog.oss-cn-beijing.aliyuncs.com/2022/01/13/16419648309816.png)

如果你计划独立使用`Operation`并希望它异步执行，那还有额外的工作要做。具体的我们将在下一节专门阐述。

## Operation Queues

1. `Operation Queues`是基于`GCD`的高层抽象。
2. 使用`Operation Queues`，才能发挥`Operations`的真正实力。你无需手动调用`start`方法，取而代之的是将其添加到`Queue`中。
3. `Operation Queues`是以面向对象的方式管理那些需要异步执行的任务。

### 使用Operation Queues

如下图，我们以`Block`的方式创建了两个`Operation`，并将他们添加到`Queue`中。之后这两个`Operation`均在后台线程开启并执行。当我们把`Operation`添加到`Queue`中，它就会尽快执行。
![](http://images-for-blog.oss-cn-beijing.aliyuncs.com/2022/01/13/16419684109100.png)

如果你想顺序的执行多个任务，只需将`maxConcurrentOperationCount`设置为`1`。如下：
![](http://images-for-blog.oss-cn-beijing.aliyuncs.com/2022/01/13/16419685107816.png)

`maxConcurrentOperationCount`限制了`Queue`中的`Operation`并发数量，默认值为`-1`，这意味着由系统决定最大并发数量。

### 依赖管理

如下图，我们可以通过在两个任务之间添加依赖来形成顺序执行的效果。
![](http://images-for-blog.oss-cn-beijing.aliyuncs.com/2022/01/13/16419689350663.png)

### 使用Operations Queue实现Dispatch Group

在之前的部分，我们使用`GCD dispatch group`实现了等待多个任务完成后继续之前的任务的效果。这里我们也可以使用`Operation Queue`来实现。
![](http://images-for-blog.oss-cn-beijing.aliyuncs.com/2022/01/13/16419700621163.png)

如上图，我们有3个任务，并希望他们并发执行，完成后再执行其他任务。这里我们做了如下工作：
1. 创建一个`Operation Queue`
2. 创建3个任务
3. 继续创建了一个任务，它在之前的任务完成后才能执行
4. 将`blockOperations4`的依赖设为之前的3个任务
5. `waitUntilFinished`会阻塞当前线程，直到所有任务完成。这里我们传入`false`。

### 相比 GCD，Operation Queues的优点

1. 相比GCD，可以很方便的设置任务的依赖
2. 支持使用KVO监听任务的执行状态
3. 任务支持暂停，取消。而使用GCD提交的任务，无法取消
4. 更方便的控制并发数量