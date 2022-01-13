---
layout: post
title: 【译】Swift并发编程二
description: 这里是`Swift并发编程系列`第二篇。主要包含`Dispatch Group`相关内容。
category:
  - iOS
tags:
  - Swift
  - GCD
  - Thread
---

> 原文地址：https://ali-akhtar.medium.com/concurrency-in-swift-grand-central-dispatch-part-2-1b0b025ee381

![并发于并行的区别](http://images-for-blog.oss-cn-beijing.aliyuncs.com/2022/01/13/0dawoyzkxm3red8.png)

这里是`Swift并发编程系列`第二篇（共四篇：[一]({{ site.url }}/2021/12/30/concurrency-in-swift-1/)）。主要包含`Dispatch Group`相关内容。

`Dispatch Group`有以下特点：

1. `Dispatch Group`可以阻塞线程，等到一个或多个任务完成后继续向下。你可以使用该特性，完成需要等待其他任务完成后才能执行的任务。

2. 在分派多个任务来计算一些数据之后，您可以使用`Dispatch Group`来等待这些任务，然后再完成后处理结果，尽管它们也可能运行在不同的队列上。
3. `Dispatch Group` 可以完成任务的聚合同步

## 使用`wait`

![1_Jc7svozjacuwSypIDUJmGA](http://images-for-blog.oss-cn-beijing.aliyuncs.com/2022/01/05/1jc7svozjacuwsypidujmga.png)

如上图，我们做了如下工作：
1. 创建了一个标签为“ com.company.app.queue”自定义并发队列
2. 创建了一个调度组
3. 使用 `group.enter()` 手动进入组 注意：对此函数的调用必须与 `group.leave()` 平衡，否则应用程序将崩溃
4. 通过`queue.async`执行的任务 1 ，它将立即返回并且 GCD 将开始并发运行此任务
5. 再次使用 `group.enter()` 手动已进入组
6. 通过`queue.async`执行任务 2，同样，它会立即返回并且 GCD 将开始并发运行此任务
7. `group.wait()` 将阻塞这里的任何线程，直到所有上述任务都完成为止（所以不要在主线程上使用这个函数），因为我们在主线程上调用了这个方法，它将阻塞主线程。

为了避免阻塞主线程，我们可以在后台线程进行等待，像这样：
![1_ip0QHOMFYKx7KRXmGuf_Fg](http://images-for-blog.oss-cn-beijing.aliyuncs.com/2022/01/05/1ip0qhomfykx7krxmguffg.png)


## 使用`notify`

在上一节，我们使用队列异步派发来避免阻塞主线程，但这看上去并不是最优解。因为它始终会阻塞一个线程。幸运的是，GCD提供了更优的方案，使用`notify`你可以在所有任务完成后获得通知：
![1*5WIff85pMpflAIxiZE6dbw](http://images-for-blog.oss-cn-beijing.aliyuncs.com/2022/01/05/15wiff85pmpflaixize6dbw.png)

在下面的例子中，任务1和任务3完成后会打印`#3 finished`，但它不会等待任务2完成。因为任务2并没有在`group`中执行。

![1*bGxd1KyZtlHUiVzd4kWi4Q](http://images-for-blog.oss-cn-beijing.aliyuncs.com/2022/01/05/1bgxd1kyztlhuivzd4kwi4q.png)
![1*4PDFt2LTYh3YFzUT7N3MSA](http://images-for-blog.oss-cn-beijing.aliyuncs.com/2022/01/05/14pdft2ltyh3yfzut7n3msa.png)

在我们的改造后，`#3 finished`会在所有任务执行完成后再输出：
![1*cfssrZ3t9pB5YH4aBg3tmg](http://images-for-blog.oss-cn-beijing.aliyuncs.com/2022/01/05/1cfssrz3t9pb5yh4abg3tmg.png)

`DispatchGroup.notify(qos:flags:queue:execute:)`接收一个参数`queue`来决定完成后的任务将在哪个队列执行。若我们指定主队列，它将在主线程执行：
![1*l1BHDTVa__M2zyZy_5leeg](http://images-for-blog.oss-cn-beijing.aliyuncs.com/2022/01/05/1l1bhdtvam2zyzy5leeg.png)


## barrier

1. `Dispatch barriers`是一组和`并发队列`一起工作的函数，提供类似`串行`执行任务的功能。
2. 当您将DispatchWorkItem（任务）提交到调度队列时，您可以设置标志以指示它应该是特定时间在指定队列上执行的唯一任务。这意味着在`Dispatch barrier`任务之前提交到队列的所有任务必须在它开始执行之前完成。

![](http://images-for-blog.oss-cn-beijing.aliyuncs.com/2022/01/05/16413532114079.jpg)

对于上面的例子：
1. 创建一个并发队列
2. 异步派发了任务1和任务2
3. 此时，上面的两个任务会并发执行。这时我们使用`barrier`作为标志添加任务3，它将在之前的任务全部完成后才开始执行，并且在任务3执行过程中，不会有其他任务执行。
4. 继续添加任务4，它将在`barrier`任务之后执行

结果如下：
![](http://images-for-blog.oss-cn-beijing.aliyuncs.com/2022/01/05/16413536631143.jpg)

我们使用`barrier`派发的任务在并发队列中独立的以`串行`方式执行了。就像下面这样：
![](http://images-for-blog.oss-cn-beijing.aliyuncs.com/2022/01/05/16413537557429.jpg)

### 读写问题

如果你在多个线程进行读写操作，你一定会碰到[`读写问题`](https://en.wikipedia.org/wiki/Readers%E2%80%93writers_problem)。而使用`Dispatch barrier`可以很好的实现多读单写模型：

```swift
class ReadWriteSolution {
  /// 依赖的并发队列
  let queue = DispatchQueue(label: "com.queue.rw", attributes: .concurrent)
  
  private var _file = 0
  var file: Int {
    get {
      // 读操作需要同步返回
      queue.sync {
        return _file
      }
    }
    set {
      // 写操作和其他的写或读需要分开
      queue.async(flags: .barrier) { [self] in
        self._file = newValue
      }
    }
  }
}
```