---
layout: post
title: 理解Cabbage框架的基础设计
category:
  - iOS
tags:
  - AVFoundation
---

`Cabbage`是基于`AVFoundation`封装的音视频处理框架。它将`AVFoundation`中零散的`API`通过适当的抽象，达到了易用、易扩展的目的。该篇是学习该框架系列文章中的一篇，主要研究其中的数据结构。

## AVFoundation

`AVFoundation`作为`Cabbage`的基础，了解其数据结构有助有后续的深入。对于这部分的熟练程度，决定了后面理解`Cabbage`的难易。下面通过类图的方式，和大家一起学习。

### 数据源

![](https://images-for-blog.oss-cn-beijing.aliyuncs.com/2020/08/21/15973730309246.jpg)

`AVAsset`是一个抽象类，代表了一个媒体资源，如一个视频或一个音频。通过该类，我们可以获取一个媒体资源的相关信息，如时长、轨道等。由于是一个抽象类，所以我们不会直接使用该类。

`AVURLAsset`是`AVAsset`的一个子类，负责从指定 URL 中加载媒体资源。最常使用的也就是它。`AVAsset`提供的有一个便利构造`+assetWithURL:`，实际上返回的就是`AVURLAsset`的实例。

我们通过 URL 初始化一个`AVAsset`对象后，其大部分属性值并没有准备好，若访问了一个未准备好的属性（比如时长）会触发`AVAsset`去更新这些值，默认这个行为是会阻塞当前线程的。为了避免阻塞，我们需要通过`AVAsynchronousKeyValueLoading`协议去异步加载:

```c
NSURL *url = <#A URL that identifies an audiovisual asset such as a movie file#>;
AVURLAsset *anAsset = [[AVURLAsset alloc] initWithURL:url options:nil];
NSArray *keys = @[@"duration"];

[asset loadValuesAsynchronouslyForKeys:keys completionHandler:^() {
    NSError *error = nil;
    AVKeyValueStatus tracksStatus = [asset statusOfValueForKey:@"duration" error:&error];
    switch (tracksStatus) {
        case AVKeyValueStatusLoaded:
            [self updateUserInterfaceForDuration];
            break;
        case AVKeyValueStatusFailed:
            [self reportError:error forAsset:asset];
            break;
        case AVKeyValueStatusCancelled:
            // Do whatever is appropriate for cancelation.
            break;
   }
}];
```

`AVComposition`代表了一个媒体作品，它是由其他媒体资源组合而成。是不可变对象。在进行视频编辑时，我们使用它的可变子类`AVMutableComposition`。比如，资源级别的操作：
![-w445](https://images-for-blog.oss-cn-beijing.aliyuncs.com/2020/08/21/15973768250016.jpg)

轨道级别的操作：
![-w429](https://images-for-blog.oss-cn-beijing.aliyuncs.com/2020/08/21/15973768791161.jpg)
在插入轨道后，会返回一个`AVMutableCompositionTrack`对象，我们可以利用它编辑该轨道：
![-w313](https://images-for-blog.oss-cn-beijing.aliyuncs.com/2020/08/21/15973786049136.jpg)
可以看到在编辑轨道时，要求的数据也是轨道类型`AVAssetTrack`。它代表了一个音轨或一个视频轨。

### 导出、播放以及生成快照

有了数据，就可以进行接下来的操作了。比如导出、播放以及生成快照。依然通过类图的方式学习一下。

![](https://images-for-blog.oss-cn-beijing.aliyuncs.com/2020/08/21/15973857650735.jpg)

先看中间的`AVAssetExportSession`。它负责将一个`AVAsset`进行导出，在指定路径下生成媒体文件。在导出之前我们可以设置导出的质量、文件格式以及是否为网络传输进行优化等。导出过程中，可以通过`progress`获取导出进度，以及通过`-cancelExport`方法取消。

`AVPlayerItem`代表了一个播放源，可以送入`AVPlayer`进行播放。也许你会有个疑问：既然`AVAsset`已经可以指定一个媒体文件了，为什么不能直接使用它进行播放？这是因为为了支持，同一个媒体文件可以同时被多个播放器进行播放。`AVPlayerItem`存储了多个播放状态。
![Playing the same asset in different ways](https://developer.apple.com/library/archive/documentation/AudioVideo/Conceptual/AVFoundationPG/Art/playerObjects_2x.png)

`AVAssetImageGenerator`是一个视频快照生成器，用于截取视频中特定时间（序列）的快照（序列）。

如果你想在这些过程中对视频或音频做自定义操作，那么就需要视频处理类（右边）和音频处理类（左边）的帮助了。

`AVAudioMix`是一个混音器，负责对音频进行调节。其中，一个`AVAudioMixInputParameters`具体负责一条音轨，通过`trackID`进行关联。默认情况下，我们可以使用它的可变子类进行音量调节，支持突变和渐变。若需要更复杂的操作，可以提供`audioTapProcessor`。

`AVVideoComposition`是一个视频混合器配置，记录者视频画面合成的参数。比如设置帧的持续时长、画布的大小等。

`AVVideoCompositionInstruction`代表了对一个时间段内的进一步调节，通过`timeRange`进行关联。比如设置该时间段内画布的背景色；对于一个时间段，可能需要处理多个视频轨，这里通过`AVVideoCompositionLayerInstruction`代表对每一个视频轨的处理，通过`trackID`进行关联。可通过其可变子类`AVMutableVideoCompositionLayerInstruction`设置`transform`、`opacity`或者`crop`。

`AVVideoCompositionCoreAnimationTool`是对`CoreAnimation`的支持，可以将`CALayer`及其支持的动画引入到视频画面的合成当中来。使用它有两种方式，一是将`Layer`内容绘制到独立的轨道中参与画面合成；二是将视频帧绘制到`Layer`上，再使用`Layer`的内容生成视频帧。需要注意的是：这种方式只适用于导出，播放时需要使用`AVSynchronizedLayer`。

`AVVideoCompositing`是一个视频混合器，它是一个协议，对视频画面的混合流程进行了规范。你可以提供自己的实现，接管多个轨道视频画面的混合。在需要进行混合时，它会收到`-startVideoCompositionRequest:`消息，并传递一个`AVAsynchronousVideoCompositionRequest`request 参数，通过这个参数，读取画布大小，轨道 id，每个轨道对应的视频画面，然后通过`finishXXX`完成一次混合。

### 小结

好了，视频编辑涉及到的类及其职责暂时到这里。总体来说，`AVFoundation`所支持的数据源单一，比较容易理解。但是在表示处理过程时，相对细致和复杂，毕竟这个过程就毕竟复杂。这里用一副图来进行一个小结：
![](https://images-for-blog.oss-cn-beijing.aliyuncs.com/2020/08/21/15973943923861.jpg)

## Cabbage

`Cabbage`在数据的组织上进行了更高层次的抽象，体现了`Swift`面向协议的特点。

### 基础协议

- `TimeRangeProvider` 定义了时间范围

#### 视频相关

- `VideoCompositionTrackProvider` 定义了一个视频作品中视频轨的来源
- `VideoCompositionProvider` 定义了图像数据的处理节点，它只有一个方法`applyEffect(to:at:renderSize:)`，接收源图像数据，并返回一个图像数据。但是这里的命名实在不理解 😂。
- `VideoProvider`并没有定义新的功能，只是将`TimeRangeProvider`、`VideoCompositionTrackProvider`、`VideoCompositionProvider`三者累加。
- `TransitionableVideoProvider` 在`VideoProvider`基础上继续累加，并添加了获取视频转场信息的能力。
- `VideoTransition` 定义了视频转场的必要信息以及转场时画面渲染规范。
- `VideoConfigurationProtocol` 也定义了图像数据的处理节点，和`VideoCompositionProvider`不同的是，它接收的参数多一个`timeRange`参数

#### 音频相关

- `AudioCompositionTrackProvider`定义了一个视频作品中音频轨的来源
- `AudioMixProvider`定义了声音数据的处理节点，这里的命名依然不理解 😂。
- `AudioProvider`并没有定义新的功能，只是将`TimeRangeProvider`、`AudioCompositionTrackProvider`、`AudioMixProvider`三者累加。
- `TransitionableAudioProvider`在`AudioProvider`基础上继续累加，并添加了获取音频转场信息的能力。
- `AudioTransition`定义了音频转场的必要信息以及转场时声音渲染规范。
- `AudioProcessingNode`定义了音频处理节点
- `AudioConfigurationProtocol`是`AudioProcessingNode`、`NSCopying`的累加，并没有添加其他功能

### 数据源

这里的数据源是指用于合成最终视频的原始资源媒体文件。`AVFoundation`中只支持`AVAsset`类型的资源，经过这里的抽象，`Cabbage`支持了图片等多种类型的原始资源。

首先，`ResourceTrackInfoProvider`协议定义了一个`ResourceTrackInfo`的来源。`ResourceTrackInfo`描述了从原始资源的`track`中选取的片段，以及在最终作品中的播放时长。

令我不解的是，这个协议还提供了另外一个获取图片的接口`image(at:renderSize:)`。似乎有点混乱。

其次，`Resource`是原始资源媒体的抽象基类，遵循`ResourceTrackInfoProvider`协议。它定义了原始资源的基本属性（如：时长、选中范围、缩放的时长、分辨率）和资源加载相关的方法。

基于这个抽象基类，其具体的子类代表不同的资源类型。如：

#### `ImageResource` 单帧图片类型

根据图片来源不同，又可以通过不同子类进行加载。如：

- `PHAssetImageResource` 代表了来源于`Phots`框架中图片。
- `AVAssetReaderImageResource` 代表了来源于`AVAssetReader`读取的图片，它支持`AVVideoComposition`配置。
- `AVAssetReverseImageResource` 也代表了来源于`AVAssetReader`读取的图片，但是是倒放。

#### `AVAssetTrackResource` 多帧图片类型

这是`AVFoundation`中默认支持的类型。在此之上，根据图片来源不同，又通过不同子类进行加载。如：

- `PHAssetTrackResource` 代表了`PHImageManager`中`requestAVAsset`的方式。
- `PHAssetLivePhotoResource`代表了`PHImageManager`中`requestLivePhoto`的方式。

总体来说，`Cabbage`中组织数据源的方式就是这样，用一张类图来小结下：

![](https://images-for-blog.oss-cn-beijing.aliyuncs.com/2020/08/21/15979862760656.jpg)

### 中间状态存储

从原始资源到最终播放或导出的视频，中间可能需要多次修改相关配置，这些配置需要有地方存储，下面这些对象都是于此相关的。

`TrackItem` 表示可编辑的一段资源。该段资源的配置也存储在这里。比如视频画面的填充模式、渲染大小、transform、透明度以及自定义的遵循`VideoConfigurationProtocol`协议的配置列表，这些信息通过`VideoConfiguration`类存储，被`TrackItem`引用。音频支持音量配置以及自定义的遵循`AudioConfigurationProtocol`协议的配置列表。另外视频的转场信息（`VideoTransition`）和音频的转场信息（`AudioTransition`）也都存储在`TrackItem`类中。

`Timeline`是最终视频的代表，它由所有视频、音频的片段、浮层、配音等组合而成。还可以设置最终视频的画布大小以及背景颜色。

### 生产线

所有的资源及配置准备好后，会通过`Timeline`传入`CompositionGenerator`进行统一装配。形成最终的播放、导出以及快照实例。

前面在`AVFoundation`一节中提到过`AVVideoCompositing`协议，它是视频混合器。`Cabbage`实现了自己的混合器`VideoCompositor`，内部通过`CoreImage`对视频进行混合。实现自己的混合器是支持除`AVAsset`之外资源中的重要一环。

为了更方便的支持`VideoCompositor`，`Cabbage`还实现了自己的`VideoCompositionInstruction`和`VideoCompositionLayerInstruction`分别对标`AVFoundation`中的`AVVideoCompositionInstruction`和`AVVideoCompositionLayerInstruction`。

音频处理方面，`Cabbage`实现了自己的`MTAudioProcessingTap`，通过`AudioProcessingTapHolder`进行管理。并通过`AudioProcessingChain`和`AudioProcessingNode`支持了链式处理。

### 小结

可以看到，`Cabbage`从开始的数据源表示，到最后的合成处理设计的相对复杂。当然带来的好处也是显而易见的，就是容易扩展。下面附上相关类的总览图：
![cabbage-structure](https://images-for-blog.oss-cn-beijing.aliyuncs.com/2020/08/21/cabbagestructure.png)

## 总结

该篇对`AVFoundation`和`Cabbage`中视频编辑相关的类和职责进行了简单梳理，这对于后续的分析大有裨益。之后继续探索相关细节，共勉！

## 参考

1. [AVFoundation Programming Guide](https://developer.apple.com/library/archive/documentation/AudioVideo/Conceptual/AVFoundationPG/Articles/00_Introduction.html)
2. [Cabbage](https://github.com/VideoFlint/Cabbage)
