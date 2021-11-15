# 理解设计模式

为了写出更`优雅`的代码，个人感觉设计模式是必过的一道坎。说它是道坎，是因为之前读设计模式居然**看不懂**😂...最近重新拾起设计模式，居然有种豁然开朗的感觉，不知怎么滴就通了。今天跟大家聊聊我的小小心得。

## 为什么说设计模式可以使你的代码更优雅

在正式开始之前，有必要先明确下设计模式的重要性。避免第一次接触设计模式感觉吃力后来就放弃，再也“拾不起来”的情况。

我们的项目中有个类似抖音首页的页面（视频Feed流，下面称为`VideoFeedViewController`），但这个页面的数据可能来自多个接口，所以就有了下面这些代码：

```objc
/// 加载第一页的视频列表
- (void)loadInitVideos {
    NSString *api = nil;
    if (首页列表）{
        api = 推荐视频接口
    }
    else if (关注列表) {
        api = 关注视频列表接口
    }
    else if (个人作品列表) {
        api = 个人作品视频列表接口
    }
    // 发起请求
}

/// 加载更多
- (void)loadMoreVideos {
    NSString *api = nil;
    NSString *params = nil;
    if (首页列表）{
        api = 推荐视频接口
        params = xxx
    }
    else if (关注列表) {
        api = 关注视频列表接口
        params = xxx
    }
    else if (个人作品列表) {
        api = 个人作品视频列表接口
        params = xxx
    }
    // 发起请求
}
```

在可预见的将来，一旦有其他地方需要播放视频Feed流，这些地方统统需要修改。结果就是这块的代码又臭又长...及难维护。

那么如何解决？依据设计模式中的`工厂方法模式`，我们可以将Feed流获取数据的这部分逻辑抽离。（严格来说这里使用的不是真正的`工厂方法模式`，而是它的变体）具体做法如下：

1. 定义Feed流的数据源。

    ```objc
    @protocol VideoFeedDataSource
    /// 加载视频列表数据
    /// @param token 用于标志加载更多；存在token，说明是加载更多
    /// @param handler 加载视频数据为异步，通过handler回调
    - (void)loadVideosWithMoreToken:(id)token completeHandler:(void (^)(NSArray<VideoModel> *videos, NSError *error))handler;
    @end
    ```
2. 根据不同的业务逻辑定义不同的数据源类型，如：`RecommendVideoFeedDataSource`、`FollowVideoFeedDataSource`、`PersonalVideoFeedDataSource`
3. `VideoFeedViewController`依赖抽象的`VideoFeedDataSource`，不同的业务将不同的数据源对象传到到`VideoFeedDataSource`中。

这样后续添加不同的数据源，就可以播放不同的视频流。也就达到我们的`优雅`要求了。

## 如何学习设计模式

个人感觉需要克服下面几个方面：

1. 工作中接触到的业务不是很复杂
2. 遇到复杂的业务，没有思考如何去做的更好
3. 前置条件太陌生

下面展开来说下这几点原因，以及如何克服它们。

### 工作中接触到的业务不是很复杂

业务模式的复杂层度会决定你的代码量。复杂业务的实现必然需要大量的代码。而大量的编码是通向设计模式的第一步，只有复杂而庞大的编码才需要设计模式。

如果你接触到的业务本身不是很复杂，那么我建议你**抄一个应用**😏

### 遇到复杂的业务，没有思考如何去做的更好

自认为类似抖音这种应用还是比较复杂的。很荣幸，我遇到了。但是遇到后缺少思考，依然无法走到最后。

写出第一节那种大量`if-else`代码的人，我个人感觉就是缺少思考。至于如何思考，这里给出我的方式：

以当前业务为中心，梳理其实现的必要条件，并将这个必要条件以恰当的方式表现出来。

还是以上面的`VideoFeedViewController`为例。我们现在考虑实现一个视频列表需要的必要条件是什么？那就是视频数据啊。
