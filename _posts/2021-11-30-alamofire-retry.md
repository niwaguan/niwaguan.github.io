---
layout: post
title: Alamofire - RetryPolicy：你搞明白了嘛？
category:
  - iOS
tags:
  - Swift
  - Alamofire
---

`RetryPolicy`是`Alamofire`中对`RequestInterceptor`的又一满分实现。从名字就可以看出，它主要是满足请求出错后的各种重试策略。下面就一起来领略一番。

## 六大配置实现五种特性

1. `retryLimit`：控制最大重试次数。超过该次数后就不在重试。默认值`2`。
2. `exponentialBackoffBase` & `exponentialBackoffScale`：这两个参数和当前重试次数一起控制两次重试直接的时间间隔。具体计算方法为：`pow(exponentialBackoffBase, retryCount) * exponentialBackoffScale`。他们的默认值是`2`和`0.5`。你可以设置`exponentialBackoffScale`为`1`实现指数增长的时间间隔。
3. `retryableHTTPMethods`：控制哪些`HTTP`方法可以重试。默认值`[delete, get, head, options, put, trace]`。这些方法都是`幂等的`。
4. `retryableHTTPStatusCodes`：控制哪些`HTTP`响应状态码可以重试。默认值`[408（请求超时）, 500, 502, 503, 504]`。
5. `retryableURLErrorCodes`：控制哪些`系统错误状态码`可以重试。默认值：
    
    1. `backgroundSessionInUseByAnotherProcess`：后台`Session`被占用
    2. `backgroundSessionWasDisconnected`：请求处理过程中后台`Session`被挂起或退出
    3. `badServerResponse`：后台返回的错误的数据
    4. `callIsActive`：请求过程中被来电打断
    5. `cannotConnectToHost`：无法链接到主机
    6. `cannotFindHost`：无法找到主机
    7. `cannotLoadFromNetwork`：需要从网络下载数据，但又被加载策略限制在`从缓存加载`
    8. `dataNotAllowed`：WIFI和蜂窝网络都没有链接
    9. `dnsLookupFailed`：无法解析DNS
    10. `downloadDecodingFailedMidStream`：下载过程中无法解码数据
    11. `downloadDecodingFailedToComplete`：下载完成后无法解码数据
    12. `internationalRoamingOff`：需要通过漫游时，该功能未打开
    13. `networkConnectionLost`：请求过程中突然断网
    14. `notConnectedToInternet`：无法建立网络连接。首次安装应用，没有获取网络权限时会是该错误码。
    15. `secureConnectionFailed`：无法建立安全的网络连接
    16. `serverCertificateHasBadDate`：SSL证书无效或过期
    17. `serverCertificateNotYetValid`：SSL证书未生效
    18. `timedOut`：超时

若上面的配置能满足需求，我们只需将该类配置在`Session`上：

```swift
let session = Session(interceptor: Interceptor(adapters: [], retriers: [RetryPolicy()]))
```
或发起请求时配置在`Request`上：

```swift
AF.request("https://httpbin.org/get", interceptor: RetryPolicy())
```

这样，在遇到以上条件的失败请求就会自动进行重试。

## 这么丰富的配置其实现并不复杂

看到提供如此丰富的配置，你是不是认为其实现会异常复杂。其实不然。总体也就几行代码：


```swift
open func shouldRetry(request: Request, dueTo error: Error) -> Bool {
    // 判断请求方法是否支持重试
    guard let httpMethod = request.request?.method, retryableHTTPMethods.contains(httpMethod) else { return false }
    // 判断响应状态码是否支持重试
    if let statusCode = request.response?.statusCode, retryableHTTPStatusCodes.contains(statusCode) {
        return true
    } else {
        // 判断系统状态码是否支持重试
        let errorCode = (error as? URLError)?.code
        let afErrorCode = (error.asAFError?.underlyingError as? URLError)?.code
        
        guard let code = errorCode ?? afErrorCode else { return false }
        // 6. retryableURLErrorCodes
        return retryableURLErrorCodes.contains(code)
    }
}
```
这个核心逻辑就这么几行代码。就是依次判断当前请求是否满足各个状态。紧接着是该方法的调用：

```swift
open func retry(_ request: Request,
                for session: Session,
                dueTo error: Error,
                completion: @escaping (RetryResult) -> Void) {
    // 判断重试次数以及其他3个条件
    if request.retryCount < retryLimit, shouldRetry(request: request, dueTo: error) {
        // 需要重试时，计算重试间隔
        completion(.retryWithDelay(pow(Double(exponentialBackoffBase), Double(request.retryCount)) * exponentialBackoffScale))
    } else {
        completion(.doNotRetry)
    }
}
```

加上这个适配方法，我们可以配置的参数都用上了。这些就是整个`RetryPolicy`的全貌。so easy嘛😂~~

## 有两点需要注意

### 为什么`retryLimit`这个配置要单独拎出来判断

在适配方法中，我们发现这里是先判断了`retryLimit`这个限制，然后再调用了核心的判断方法。为什么`retryLimit`的判断没有在核心方法中判断？

我的想法是：`retryLimit`这个条件是最容易触发`false`的，在这种情况下，可以节省一次函数调用。希望有不同见解同学评论区告诉我，一起进步😁。

### `retryableURLErrorCodes`没有覆盖到的点

虽然`retryableURLErrorCodes`这个配置支持了`notConnectedToInternet`这个错误码。但在应用首次安装，网络未授权时表现不佳。

考虑这个场景：用户第一次打开应用，弹出网络授权，然后在此状态停留。此时我们的应用并没有网络授权，然后请求会出错，根据这个重试策略，它会再次发出请求，然后还是错误....这样一直循环，知道重试次数用完。

#### 解决方案

1. 开启`URLSessionConfiguration`的`waitsForConnectivity`配置。这个配置开启后会在没有网络连接时将请求挂起，等到有链接的时候再发出。但是只能在`iOS 11.0`之后使用。（话说还有多少同学在跟`11`之前的版本斗争？留个言我同情你下😂）
2. 自己实现`RequestInterceptor`。在内部监听网络状态，根据状态决定是否重试。这个方案交给各位同学啦。


## 总结

今天学习了`RetryPolicy`的使用及其实现。还发现了两个有趣的小问题。下节课再见。拜拜👋🏻。