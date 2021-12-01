---
layout: post
title: 深入理解Cabbage框架
category:
  - iOS
tags:
  - AVFoundation
---

在[上篇--理解 Cabbage 框架的基础设计]({{ site.url }}/2020/08/21/cabbage-arch/)一问中我们梳理了`Cabbage`的基本类结构。本篇主要分析相关细节实现。

## 体验繁琐

`AVFoundation`提供了一套很强大的`api`，但是使用起来却很繁琐。不体验`AVFoundation`的繁琐，就无法真正理解`Cabbage`的简单。短视频应用中，最常见的需求是，将用户选定的多个视频片段拼接在一起，然后播放。我们就通过一个小例子感受下`AVFoundation`。

```swift
/// 根据给定资源创建视频作品
/// 注意，这里对大量的<Optional>类型使用了强制拆包，实际不建议这样做
func makeComposition(from assets: [AVAsset]) -> AVComposition {
    // 创建可变作品
    let composition = AVMutableComposition()
    // 添加视频轨
    let videoTrack = composition.addMutableTrack(withMediaType: .video, preferredTrackID: kCMPersistentTrackID_Invalid)!
    // 添加音轨
    let audioTrack = composition.addMutableTrack(withMediaType: .audio, preferredTrackID: kCMPersistentTrackID_Invalid)!
    // 还可以添加其他类型轨道，如字幕等...

    var cursor: CMTime = .zero
    for asset in assets {
        let duration = asset.duration
        // 查找资源中的「视频轨」 添加到作品视频轨中
        let video = asset.tracks(withMediaType: .video).first!
        try! videoTrack.insertTimeRange(CMTimeRange(start: .zero, duration: duration), of: video, at: cursor)
        // 查找资源中的「音频轨」 添加到作品音频轨中
        let audio = asset.tracks(withMediaType: .audio).first!
        try! audioTrack.insertTimeRange(CMTimeRange(start: .zero, duration: duration), of: audio, at: cursor)
        cursor = CMTimeAdd(cursor, duration)
    }

    return composition
}
```

通过上面的代码，我们可以完成拼接工作。是不是有种感觉：并不是很麻烦啊？实际跑起来的时候，你会发现这效果并不是你想要的。比如：图像方向不对、填充模式不对、音乐声音无法控制。下面我们借助`AVVideoComposition`来修复这些问题

```swift
/// 根据给定资源创建播放源
func makePlayerItem(from assets: [AVAsset]) -> AVPlayerItem {
    // 创建可变作品
    let composition = AVMutableComposition()
    // 添加视频轨
    let videoTrack = composition.addMutableTrack(withMediaType: .video, preferredTrackID: kCMPersistentTrackID_Invalid)!
    // 视频轨控制指令
    let videoTrackLayerInstruction = AVMutableVideoCompositionLayerInstruction(assetTrack: videoTrack)
    // 添加音轨
    let audioTrack = composition.addMutableTrack(withMediaType: .audio, preferredTrackID: kCMPersistentTrackID_Invalid)!
    // 音频轨控制参数
    let audioTrackInputParamters = AVMutableAudioMixInputParameters(track: audioTrack)
    // 还可以添加其他类型轨道，如字幕等...

    // 视频渲染大小
    let renderSize = CGSize(width: 1920, height: 1080)

    var cursor: CMTime = .zero
    for asset in assets {
        let range = CMTimeRange(start: cursor, duration: asset.duration)
        // 查找资源中的「视频轨」 添加到作品视频轨中
        let video = asset.tracks(withMediaType: .video).first!
        try! videoTrack.insertTimeRange(CMTimeRange(start: .zero, duration: range.duration), of: video, at: cursor)
        // 设置视频图像的transform，来完成图像的移动及缩放
        // 这里以填充模式下为例
        var transform = video.preferredTransform
        var size = video.naturalSize.applying(transform)
        let backup = size
        if (size.width < size.height) {
            // 纵向填充
            let rate = renderSize.height / size.height
            size.width = rate * size.width
            size.height = renderSize.height
        } else {
            // 横向填充
            let rate = renderSize.width / size.width
            size.height = rate * size.height
            size.width = renderSize.width
        }
        // 计算最终的transform，先平移，再缩放。这里的变换都是以左上角为原点
        transform = transform
            .translatedBy(x: (renderSize.width - size.width) / 2 , y: (renderSize.height - size.height) / 2)
            .scaledBy(x: size.width/backup.width, y: size.height/backup.height)
        videoTrackLayerInstruction.setTransform(transform, at: cursor)

        // 查找资源中的「音频轨」 添加到作品音频轨中
        let audio = asset.tracks(withMediaType: .audio).first!
        try! audioTrack.insertTimeRange(CMTimeRange(start: .zero, duration: range.duration), of: audio, at: cursor)
        let ramp = CMTime(seconds: 1, preferredTimescale: 600)
        // 渐入
        audioTrackInputParamters.setVolumeRamp(fromStartVolume: 0, toEndVolume: 1.0, timeRange: CMTimeRange(start: cursor, duration: ramp))
        // 渐出
        audioTrackInputParamters.setVolumeRamp(fromStartVolume: 1.0, toEndVolume: 0, timeRange: CMTimeRange(start: CMTimeSubtract(CMTimeAdd(cursor, range.duration), ramp), duration: ramp))

        cursor = CMTimeAdd(cursor, range.duration)
    }

    // 视频合成控制
    let videoComposition = AVMutableVideoComposition()
    // 指定画布大小，必须
    videoComposition.renderSize = CGSize(width: 1920, height: 1080)
    // 指定帧时长，必须
    videoComposition.frameDuration = CMTime(value: 1, timescale: 30)
    // 关联视频控制指令
    let instruction = AVMutableVideoCompositionInstruction()
    instruction.timeRange = CMTimeRange(start: .zero, end: cursor)
    // 添加一个红色背景，方便测试
    instruction.backgroundColor = UIColor.red.cgColor
    instruction.layerInstructions = [ videoTrackLayerInstruction ]
    videoComposition.instructions = [ instruction ]

    // 音频合成控制
    let audioMix = AVMutableAudioMix()
    audioMix.inputParameters = [ audioTrackInputParamters ]

    // 生成playerItem
    let item = AVPlayerItem(asset: composition)
    item.videoComposition = videoComposition
    item.audioMix = audioMix

    return item
}
```

这次实现的效果还是可以的，视频画面方向正确，位置居中，并且填充，声音在存在渐变。可以看到，满足这一小小需求就需要这么一大坨代码。要知道，这里还没有考虑其他的填充模式，没有考虑转场等等...

## 化繁为简

还是上面的需求，我们尝试用`Cabbage`来完成。

```swift
func makePlayerItem(from assets: [AVAsset]) -> AVPlayerItem {
    // 根据资源生成使用track item
    let items = assets.map { (asset) -> TrackItem in
        let resource = AVAssetTrackResource(asset: asset)
        let item = TrackItem(resource: resource)
        item.videoConfiguration.contentMode = .aspectFit
        return item
    }
    // 构造timeline
    let timeline = Timeline()
    timeline.videoChannel = items
    timeline.audioChannel = items
    try! Timeline.reloadVideoStartTime(providers: timeline.videoChannel)
    try! Timeline.reloadAudioStartTime(providers: timeline.audioChannel)
    // 生成player item
    let compositionGenerator = CompositionGenerator(timeline: timeline)
    let playerItem = compositionGenerator.buildPlayerItem()
    return playerItem
}
```

哇卡卡，原来的一坨变成现在几行代码，简直不要太爽。那么接下来的任务就是扒开表层，深入内部，学习这种封装思想。

### 从表象开始

参考上面的代码，作为`Cabbage`的使用方来说，我们需要完成 3 个步骤：

1. 准备资源
2. 使用资源构造`Timeline`
3. 把`Timeline`塞到`CompositionGenerator`中进行加工

可以这样理解，前两步是数据的记录配置阶段，属于数据的产生；而最后一步通过配置获取结果，属于数据的消费。在[理解 Cabbage 框架的基础设计]({{ site.url }}/2020/08/21/cabbage-arch/)中，我大致梳理了相关的类及其职责，那么通过深究`CompositionGenerator`的消费过程，可以使你更加明白。

### 深陷其中

`CompositionGenerator`提供的接口主要有三个：

1. `public func buildPlayerItem() -> AVPlayerItem {}` 获取用于播放的播放条目
2. `public func buildImageGenerator() -> AVAssetImageGenerator {}` 获取用于生成快照的快照生成器
3. `public func buildExportSession(presetName: String) -> AVAssetExportSession? {}` 获取用于导出文件的导出器

而这三个过程又极其的相似：

1. 生成`AVComposition`
2. 生成`AVVideoComposition`用于控制视频画面
3. 生成`AVAudioMix`用于混音（生成快照时不需要）
4. 使用上面的对象分别生成配置`AVPlayerItem`、`AVAssetImageGenerator`以及`AVAssetExportSession`并返回。

如下：

![](https://images-for-blog.oss-cn-beijing.aliyuncs.com/2020/08/30/15983316382095.jpg)

自然，这里的重点就落在了如何生成前三步的对象了。

#### buildComposition

该阶段负责读取`timeline`中的资源，将其插入到`AVMutableComposition`合适的轨道中。这里主要分为 4 个步骤：

1. 处理视频资源
2. 处理音频资源
3. 处理浮层（如贴纸）资源
4. 处理额外的音频（如配音）资源

整体过程比较清晰，下面说下我认为需要注意点。

##### 视频音频资源的轨道安排

视频资源和音频资源的轨道都使用了`A/B`轨模式（在两条轨道交叉放入各个资源的内容）。比如 3 个资源的轨道分布如下：

![-w693](https://images-for-blog.oss-cn-beijing.aliyuncs.com/2020/08/30/15986885818945.jpg)

原因是，在`AVMutableCompositionTrack`中插入资源过程中，指定的时间点存在内容时，会根据该点将内容一分为二，后面的内容向后平移，新的内容插入。这个行为导致在一个轨道的同一个时间上无法同时存在两段内容，也就无法做类似转场的需求。所以`A/B`轨可以避开这个问题。

##### 浮层内容的安排

为了节省资源，避免插入过多的轨道。在浮层的处理过程中，会尽力复用之前的轨道。但是，这里我找出了一个`bug`，导致最终复用的目的未达到。一起看代码吧，注释写的比较清楚：

```swift
// 记录处理浮层过程中产生的所有轨道
var overlaysTrackIDs: [Int32] = []
timeline.overlays.forEach { (provider) in
    for index in 0..<provider.numberOfVideoTracks() {
        // 确定浮层在作品中的track。重点⚠️
        let trackID: Int32 = {
            // 遍历已经产生的轨道，寻找可以复用的
            if let trackID = overlaysTrackIDs.first(where: { (trackID) -> Bool in
                // 根据id获取轨道实例
                if let track: AVCompositionTrack = composition.track(withTrackID: trackID) {
                    // 遍历track的所有片段
                    for segment in track.segments {
                        /** p: provider
                         |~~p1~~|           |~~~~~~p2~~~~~~|                  |~p3~|
                                  |--segment1--|        |------segment2------|
                         |____________________________track_________________________|
                         */
                        // p1的情况，已存在的片段（segment1）起点在当前资源（p1）结尾的后面。说明找到了空余位置，该轨道可以复用
                        // 此时，该资源的起点有没有可能和之前片段（X）有交集？不会，因为在遍历（X）时被下面的判断过滤掉。
                        if segment.timeMapping.target.start > provider.timeRange.end {
                            break
                        }
                        // p3的情况，已存在的片段（segment2）结尾在当前资源（p3）的前面。此时需要观察下一个片段，因为
                        // 可能会和下一个的片段重合
                        if segment.timeMapping.target.end < provider.timeRange.start {
                            continue
                        }
                        // p2segment1交叉的情况，当前资源和已有资源片段有交集，这样的不能复用
                        if !segment.isEmpty {
                            let intersection = provider.timeRange.intersection(segment.timeMapping.target)
                            if intersection.duration.seconds > 0 {
                                return false
                            }
                        }
                        // 这句应该移动到for循环的外面，来保证p1情况的break，使当前轨道可以复用
                        return true
                    }
                    // return 应该在这里
                }
                // 未找到轨道
                return false
            }) {
                return trackID;
            }
            return generateNextTrackID()
        }()
        // 根据上面获取到的轨道id，向作品中插入，并建立轨道和资源的映射
        if let compositionTrack = provider.videoCompositionTrack(for: composition, at: index, preferredTrackID: trackID) {
            let info = TrackInfo.init(track: compositionTrack, info: provider)
            overlayTrackInfo.append(info)
        }

        if !overlaysTrackIDs.contains(trackID) {
            overlaysTrackIDs.append(trackID);
        }
    }
}
```

#### buildVideoComposition

该阶段，主要生成`AVVideoComposition`对象，对视频进行控制。分为 2 个主要步骤：

1. 按时间段生成`instructions`控制指令
2. 使用自定义的视频混合器

##### instructions 控制指令

在`buildComposition`阶段，处理视频资源时，会记录`轨道`和`资源集`的映射，记录在`mainVideoTrackInfo`和`overlayTrackInfo`数组中。这里会遍历两个数组，生成`VideoCompositionLayerInstruction`数组。

得到`VideoCompositionLayerInstruction`数组后，经过排序，再按时间段分组。最终形成`[time: [VideoCompositionLayerInstruction]]`字典结构。每一个`key-value`组合将对应一个`VideoCompositionInstruction`。这样，就会得到控制指令数组。

##### 视频混合器

视频混合器是自定义的`VideoCompositor`。在适当时机会收到系统回调，顺序是：

1. 通过`renderContextChanged(_:)`接收到渲染上下文
2. 通过`startRequest(_:)`接收异步的混合请求，处理完之后需要回调结构
3. 通过`cancelAllPendingVideoCompositionRequests()`接收取消事件

接下来，参考下面的类图，以及调用流程图，一起分析下具体的处理过程：
![相关类图](https://images-for-blog.oss-cn-beijing.aliyuncs.com/2020/08/30/15987760400948.jpg)
![调用流程图](https://images-for-blog.oss-cn-beijing.aliyuncs.com/2020/08/30/15987794890493.jpg)

开始阶段。`renderContextChanged(_:)`将被调用，携带渲染上下参数，`VideoCompositor`只是记录，并没有多于操作。

接收混合请求。这个阶段是真正的混合阶段，牵扯到的方法较多，并且比较名称比较相似，具体可参考调用流程图的标注。重点是`VideoCompositionInstruction`中的`apply(request:)`方法：

```swift
open func apply(request: AVAsynchronousVideoCompositionRequest) -> CIImage? {
    // 当前混合的时间点
    let time = request.compositionTime
    // 画布大小
    let renderSize = request.renderContext.size
    // 区分主轴和其他轴的指令
    var otherLayerInstructions: [VideoCompositionLayerInstruction] = []
    var mainLayerInstructions: [VideoCompositionLayerInstruction] = []

    for layerInstruction in layerInstructions {
        if mainTrackIDs.contains(layerInstruction.trackID) {
            mainLayerInstructions.append(layerInstruction)
        } else {
            otherLayerInstructions.append(layerInstruction)
        }
    }

    var image: CIImage?
    // 两个轨道时支持转场
    if mainLayerInstructions.count == 2 {
        // 确定转场的from和to
        let layerInstruction1: VideoCompositionLayerInstruction // from
        let layerInstruction2: VideoCompositionLayerInstruction // to
        if mainLayerInstructions[0].timeRange.end < mainLayerInstructions[1].timeRange.end {
            layerInstruction1 = mainLayerInstructions[0]
            layerInstruction2 = mainLayerInstructions[1]
        } else {
            layerInstruction1 = mainLayerInstructions[1]
            layerInstruction2 = mainLayerInstructions[0]
        }
        // 获取原始图片数据
        if let sourcePixel1 = request.sourceFrame(byTrackID: layerInstruction1.trackID),
            let sourcePixel2 = request.sourceFrame(byTrackID: layerInstruction2.trackID) {
            // 确定转场的from图片
            let image1 = generateImage(from: sourcePixel1)
            let sourceImage1 = layerInstruction1.apply(sourceImage: image1, at: time, renderSize: renderSize)
            if let transition = layerInstruction1.transition {
                // 确定转场的to图片
                let image2 = generateImage(from: sourcePixel2)
                let sourceImage2 = layerInstruction2.apply(sourceImage: image2, at: time, renderSize: renderSize)
                // 确定转场进度
                let transitionTimeRange = layerInstruction1.timeRange.intersection(layerInstruction2.timeRange)
                let tweenFactor = factorForTimeInRange(time, range: transitionTimeRange)
                // 对from和to两个图片应用转场
                let transitionImage = transition.renderImage(foregroundImage: sourceImage2, backgroundImage: sourceImage1, forTweenFactor: tweenFactor, renderSize: renderSize)
                image = transitionImage
            } else {
                image = sourceImage1
            }
        }
    } else {
        mainLayerInstructions.forEach { (layerInstruction) in
            // 顺次取出每一个轨道的图片
            if let sourcePixel = request.sourceFrame(byTrackID: layerInstruction.trackID) {
                // 使用layerInstruction处理该图像
                let sourceImage = layerInstruction.apply(sourceImage: CIImage(cvPixelBuffer: sourcePixel), at: time, renderSize: renderSize)
                // 若存在之前的图片，将其混合。越之前的图片，在越底部
                if let previousImage = image {
                    image = sourceImage.composited(over: previousImage)
                } else {
                    image = sourceImage
                }
            }
        }
    }
    // 混合其他轨道的图片
    otherLayerInstructions.forEach { (layerInstruction) in
        if let sourcePixel = request.sourceFrame(byTrackID: layerInstruction.trackID) {
            let sourceImage = layerInstruction.apply(sourceImage: CIImage(cvPixelBuffer: sourcePixel), at: time, renderSize: renderSize)
            if let previousImage = image {
                image = sourceImage.composited(over: previousImage)
            } else {
                image = sourceImage
            }
        }
    }
    // 应用全局效果
    if let passingThroughVideoCompositionProvider = passingThroughVideoCompositionProvider, image != nil {
        image = passingThroughVideoCompositionProvider.applyEffect(to: image!, at: time, renderSize: renderSize)
    }

    return image
}
```

#### buildAudioMix

这个过程中，通过自定义`AVAudioMixInputParameters`的`audioTapProcessor`构建了音频混合器。整个过程在`TrackItem`的`configure(audioMixParameters:)`方法中实现，这是来自`AudioMixProvider`协议的方法：

```swift
open func configure(audioMixParameters: AVMutableAudioMixInputParameters) {
    let volume = audioConfiguration.volume
    // 设置音量
    audioMixParameters.setVolumeRamp(fromStartVolume: volume, toEndVolume: volume, timeRange: timeRange)
    // 配置audioTapProcessor
    if audioConfiguration.nodes.count > 0 {
        if audioMixParameters.audioProcessingTapHolder == nil {
            audioMixParameters.audioProcessingTapHolder = AudioProcessingTapHolder()
        }
        // 配置音频处理节点
        audioMixParameters.audioProcessingTapHolder?.audioProcessingChain.nodes.append(contentsOf: audioConfiguration.nodes)
    }
}
```

`AVMutableAudioMixInputParameters`的`audioProcessingTapHolder`属性是使用`runtime`添加的。在其`setter`方法中，`audioTapProcessor`属性得到真正的赋值。而最终自定义音频的处理，体现在`AudioProcessingTapHolder`中`audioProcessingChain`属性。`AudioProcessingTapHolder`在构建`MTAudioProcessingTap`是，会设置一个数据回调`tapProcess`，在回调中，会向外回调`audioProcessingChain`。

## 总结

总体来看，`Cabbage`的封装是成功的，它将大部分的固有流程都处理了，并将可定制化的接口都留出来了。但，通过一路的分析，我也感觉到该框架并没有达到其该有的形态，有些部分还是有点繁琐。下一篇中，我将谈谈自己的理解，并作出改进。

共勉！
