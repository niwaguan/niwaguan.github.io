---
layout: post
title: 理解WebViewJavascriptBridge框架
category:
  - iOS
tags:
  - WKWebView, WebView, WebViewJavascriptBridge
---

[`WebViewJavascriptBridge`](https://github.com/marcuswestin/WebViewJavascriptBridge)是iOS开发中混合H5页面时经常用到的三方库。使用它可以很方便的在iOS和JS之间相互调用。该篇文章将探究其所以然，主要有两个目标：

1. JS如何调用iOS的？
2. iOS如何调用JS的？

> 该篇主要分析`WKWebViewJavascriptBridge`，其他兼容性相关代码暂且不表。

# 顺藤摸瓜-从初始化开始

按照官方的教程，使用`WebViewJavascriptBridge`需要iOS、JS双端都进行初始化配置。我们就从这里入手，看看能不能找出端倪。

## iOS端

```objc
[WKWebViewJavascriptBridge bridgeForWebView:webView];
```

在引入库后，需要先使用上面的方法生成`Bridge`对象。再使用`Bridge`对象注册供JS调用的方法。

```objc
+ (instancetype)bridgeForWebView:(WKWebView*)webView {
    WKWebViewJavascriptBridge* bridge = [[self alloc] init];
    [bridge _setupInstance:webView];
    [bridge reset];
    return bridge;
}

- (void) _setupInstance:(WKWebView*)webView {
    _webView = webView;
    _webView.navigationDelegate = self; // 1
    _base = [[WebViewJavascriptBridgeBase alloc] init]; // 2
    _base.delegate = self;
}

- (void)reset {
    [_base reset];
}

/// WebViewJavascriptBridgeBase
- (void)reset { // 3
    self.startupMessageQueue = [NSMutableArray array];
    self.responseCallbacks = [NSMutableDictionary dictionary];
    _uniqueId = 0;
}
```

可以看到，整个初始化非常的简单。生成实例对象，进行必要的配置，就返回了。有几点需要注意的是：

1. 我们传入的`webView`的导航代理被设置为`Bridge`对象。所以我们需要通过`-setWebViewDelegate:`设置我们自己的代理，以接收相应回调。
2. `WebViewJavascriptBridgeBase`对象是幕后的大boss，`Bridge`对象的诸多api都依赖它。这样设计的原因是该库兼容多种类型的`WebView`，base实现了基本逻辑。
3. 重置时会清空base对象的`startupMessageQueue`、`responseCallbacks`以及`_uniqueId`。这3个属性是整个库的核心内容，后面会详细说明。

完成初始化后，下一步就是注册对应的方法了。

```objc
- (void)registerHandler:(NSString *)handlerName handler:(WVJBHandler)handler {
    _base.messageHandlers[handlerName] = [handler copy];
}
```
😂简单的不能再简单了，就是在base的`messageHandlers`中以`key-value`的方式记录下来。

## JS端

JS端的初始化也只是执行了下面一段简单的代码：

```js
function setupWebViewJavascriptBridge(callback) {
  if (window.WebViewJavascriptBridge) {
    return callback(window.WebViewJavascriptBridge);
  }
  if (window.WVJBCallbacks) {
    return window.WVJBCallbacks.push(callback);
  }
  window.WVJBCallbacks = [callback];
  var WVJBIframe = document.createElement("iframe");
  WVJBIframe.style.display = "none";
  WVJBIframe.src = "https://__bridge_loaded__";
  document.documentElement.appendChild(WVJBIframe);
  setTimeout(function () {
    document.documentElement.removeChild(WVJBIframe);
  }, 0);
}
```

从这里我们可以了解到：

1. `window.WebViewJavascriptBridge`是JS端的主要依赖对象。后续JS使用的所有api都在该对象中。
2. `window.WVJBCallbacks`是`callback`s的持有者。多次调用`setupWebViewJavascriptBridge`方法，只会记录多个`callback`。
3. 通过在`dom`上挂载`iframe`，发起加载请求，告知iOS端。

直觉告诉我这个字符串不简单：`https://__bridge_loaded_`。然而...在库中搜索，却没有发现是怎么使用的。（后来才发现自己脑子秀逗了...😂）

# 柳暗花明 - webview的代理

在iOS端初始化时，设置了`webview`的代理。这个操作绝对事出有因。于是翻了下具体实现的代理方法：
![-w970](http://images-for-blog.oss-cn-beijing.aliyuncs.com/2021/09/25/16324096456168.jpg)

乍一看还挺多。仔细捋一捋，其实没啥，90%都是在给`_webView.navigationDelegate = self`这句擦屁股...

下面红框的代码才是重中之重：
![-w964](http://images-for-blog.oss-cn-beijing.aliyuncs.com/2021/09/25/16324100541093.jpg)

1. 处理JS端的初始化事件，注入JS api依赖对象
2. 处理JS端发起的调用，刷新消息队列
3. 异常处理

# 深入浅出 - `iframe`和`-evaluateJavaScript:completionHandler:`是绝对的功臣

先上一张事件序列图，听我慢慢道来：
![webview-javascript-bridge](http://images-for-blog.oss-cn-beijing.aliyuncs.com/2021/09/25/webviewjavascriptbridge.svg)

## 被字符串秀到了

在上一部分说JS端初始化中，会通过`iframe`加载`https://__bridge_loaded_`链接，来通知iOS端。但没有在iOS端找到使用它地方！其实，是我搜索的有问题，应该搜索`__bridge_loaded_`...

在`webview`的代理中，触发了下面的方法：

```objc
- (BOOL)isBridgeLoadedURL:(NSURL*)url {
    NSString* host = url.host.lowercaseString;
    return [self isSchemeMatch:url] && [host isEqualToString:kBridgeLoaded];
}
```

可以看到，这里先校验了`scheme`，之后校验了`host`。是分开的！！

在判断是`load`过程后，就进入了`注入`阶段。对应的是`-injectJavascriptFile`方法。这里主要干了两件事：

1. 注入`window.WebViewJavascriptBridge`对象，提供JS端api。
2. 分发iOS配置的启动消息或者在JS环境没有准备好之前iOS端的调用。

## JS调用iOS

对于JS端的调用：

```js
  bridge.callHandler(
    "pickImage",
    { key: "value" },
    function responseCallback(responseData) {
      console.log("JS received response:", responseData);
    }
  );
```
其对应的调用栈为：
![-w1668](http://images-for-blog.oss-cn-beijing.aliyuncs.com/2021/09/25/16324733007185.jpg)

可以很清楚的看到：

1. `callHandler`只是将入参拼接为`message`，传入`_doSend`。另外`callHandler`也支持这种调用：`callHandler('methodName', () => {})`
2. `_doSend`是真正处理发送`message`的逻辑。在有回调的情况下，生成对应的回调id，然后使用对应id将回调函数存储在`responseCallbacks`中；同时，这个id也会使用`callbackId`作为键名插入`message`，便于iOS端处理该次JS调用后，回调到JS。这个`message`，最后会被放入`sendMessageQueue`中，在发起新的`iframe`加载（`url: https://__wvjb_queue_message__`）后会被iOS处理。

到这里，JS调用iOS这一过程中，JS端的处理算是完成了。

回到iOS端，`WKFlushMessageQueue`是重要的一步：

```objc
- (void)WKFlushMessageQueue {
    [_webView evaluateJavaScript:[_base webViewJavascriptFetchQueyCommand] completionHandler:^(NSString* result, NSError* error) {
        if (error != nil) {
            NSLog(@"WebViewJavascriptBridge: WARNING: Error when trying to fetch data from WKWebView: %@", error);
        }
        [_base flushMessageQueue:result];
    }];
}
```

这里先是执行了一段JS，成功后处理返回的结果。

这段JS`WebViewJavascriptBridge._fetchQueue();`就是获取上一步`sendMessageQueue`中的内容：

```js
function _fetchQueue() {
  var messageQueueString = JSON.stringify(sendMessageQueue);
  sendMessageQueue = [];
  return messageQueueString;
}
```

很清楚，返回JSON化的字符串。那么下一步iOS端必然会存在解析JSON的过程：
![-w1440](http://images-for-blog.oss-cn-beijing.aliyuncs.com/2021/09/25/16325517494591.jpg)

可以看到：
1. 在必要的入参判断后，第一步就是解析JSON。
2. 若存在`callbackId`，则生成响应JS调用的Block，这个Block就是向JS发送消息，内容为`@{ @"responseId":callbackId, @"responseData":responseData }`。
3. 使用`handlerName`从`messageHandlers`中取出该次调用对应的处理者，然后将该次调用的数据和生成的Block一起传入处理者。
4. iOS调用JS时，存在回调时，JS也会发送一个响应回调，就像这里的第二步。

## iOS调用JS

同样，对于iOS的调用：

```objc
[_bridge callHandler:@"reload" data:@{ @"by":@"test" } responseCallback:^(id responseData) {
    NSLog(@"call reload response: %@", responseData);
}];
```
可以跟踪到以下调用栈：
![-w544](http://images-for-blog.oss-cn-beijing.aliyuncs.com/2021/09/25/16325591287401.jpg)

我们逐个看下每个方法：

1. `-callHandler:data:responseCallback:`什么也没做，只是调用了下一步的方法。
2. `-sendData:responseCallback:handlerName:` ![-w826](http://images-for-blog.oss-cn-beijing.aliyuncs.com/2021/09/25/16325616907130.jpg)这里会生成`message`，主要包含`callbackId`和`handlerName`两个字段。和JS端拼装的消息如出一辙。紧接着就将消息传入下一个方法。需要注意的是，这里也会通过`callbackId`在`responseCallbacks`中记录回调Block

3. `-_queueMessage:`会区分初始化是否完成。未完成时，会将该次调用存储在`startupMessageQueue`中，后续初始化完成后再分发；若已经初始化完成，会直接进入下一步。
4. `-_dispatchMessage:` ![-w902](http://images-for-blog.oss-cn-beijing.aliyuncs.com/2021/09/25/16325621085434.jpg)在这里会将上一步生成的`message`转换成JSON，然后替换特殊字符，最后拼接到`WebViewJavascriptBridge._handleMessageFromObjC('%@')`中，使用`webview`的`-evaluateJavaScript:completionHandler:`执行。
5. 最后我们在看一下JS端的处理流程：
    1. `_handleMessageFromObjC`直接调用了`_dispatchMessageFromObjC`
    2. `_dispatchMessageFromObjC`会根据是否开启`dispatchMessagesWithTimeoutSafety`（默认true）来确定是否通过`setTimeout`调用`_doDispatchMessageFromObjC`。这里使用这个开关，是因为JS调用`alert, confirm, and prompt`会导致app挂起。具体没有测试出来，还请知晓的同学告知下。
    3. `_doDispatchMessageFromObjC`是这里的重头戏，和iOS端的处理逻辑类似。![-w578](http://images-for-blog.oss-cn-beijing.aliyuncs.com/2021/09/25/16325722509953.jpg)还是先JSON解析。之后根据`callbackId`生成回调函数，加上传过来的数据调用根据`handlerName`找到的处理函数。

至此，我们就完全梳理了两端相互调用的逻辑。有问题欢迎大家提问。

# 值得思考的点

## 为什么iOS端处理JS的调用时，需要使用批处理。而不是每次JS调用iOS都分别处理一次？

JS调用iOS是通过`iframe.src="https://__wvjb_queue_message__"`来触发iOS的处理流程的。频繁的设置`iframe.src`浏览器并不会及时触发更新。若每次调用iOS，单独处理，可能会造成调用丢失问题。

## 为什么JS端的支持对象`window.WebViewJavascriptBridge`需要通过JS触发，iOS注入的方式进行？

这里应该是考虑到JS通过网络加载带了的延迟问题。

## 为什么JS调用iOS时的数据都被被JSON化了？

在`WKWebView`中JS和iOS两端的数据类型会自动转换，使用JSON做中转应该不是必须的。这里可能是历史遗留问题。在从`UIWebView`更新到`WKWebView`时，这部分的逻辑保留了旧版的处理方式。