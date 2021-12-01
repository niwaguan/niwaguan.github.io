---
layout: post
title: Alamofire - 使用拦截器优雅的对接口进行授权
category:
  - iOS
tags:
  - Swift, Alamofire
---

我们在之前分析拦截器的[文章]({{ site.url }}/2021/11/26/alamofire-request-interceptor/)中提到，`Alamofire`中实现了一些比较常用的拦截器。`AuthenticationInterceptor`绝对是满分（我打的分 🤣）实现之一。今天一起来拜读一下。

> 和`AuthenticationInterceptor`类似的还有`RetryPolicy`，也可谓精辟。具体内容放在下篇展开，敬请期待。

## 面临的问题

在实际的项目中我们经常遇到的问题是：部分 API 是需要授权之后才能够访问。例如：我们获取用户信息的接口`api.xx.com/users/id`，需要在请求头中添加`Authorization: Bearer accessToken`以完成授权，否则服务器会返回`401`拒绝我们访问。这个`accessToken`会有过期时间，过期后我们需要重新获取，一般是通过登陆接口返回。后来为了减少用户登录频率，和`accessToken`一起返回的还有`refreshToken`，它的有效期会比`accessToken`稍长，可以使用它来对`accessToken`进行刷新，就可以避免用户登录操作。

> 这里涉及`OAuth2.0`以及`JWT`相关背景知识，不了解的同学自行解决哈。

那么对于上面的需求，我们客户端需要做的有哪些呢？具体如下：

1. 获取`accessToken`和`refreshToken`
2. 在后续需要授权的接口中添加请求头
3. `accessToken`过期后，使用`refreshToken`进行刷新
4. 刷新`accessToken`失败时，需要用户登录重新授权。

那么`Alamofire`为我们做了哪些？继续看 😁

## 如何解决

首先，我们可以定义一个自己的凭证（也就是后续需要用到的认证信息）：

```swift
struct OAuthCredential: AuthenticationCredential {
    let accessToken: String
    let refreshToken: String
    let userID: String
    let expiration: Date

    // 这里我们在有效期即将过期的5分钟返回需要刷新
    var requiresRefresh: Bool { Date(timeIntervalSinceNow: 60 * 5) > expiration }
}
```

其次，我们再实现一个自己的授权中心：

```swift
class OAuthAuthenticator: Authenticator {
    /// 添加header
    func apply(_ credential: OAuthCredential, to urlRequest: inout URLRequest) {
        urlRequest.headers.add(.authorization(bearerToken: credential.accessToken))
    }
    /// 实现刷新流程
    func refresh(_ credential: OAuthCredential,
                 for session: Session,
                 completion: @escaping (Result<OAuthCredential, Error>) -> Void) {
    }

    func didRequest(_ urlRequest: URLRequest,
                    with response: HTTPURLResponse,
                    failDueToAuthenticationError error: Error) -> Bool {
        return response.statusCode == 401
    }

    func isRequest(_ urlRequest: URLRequest, authenticatedWith credential: OAuthCredential) -> Bool {
        let bearerToken = HTTPHeader.authorization(bearerToken: credential.accessToken).value
        return urlRequest.headers["Authorization"] == bearerToken
    }
}

```

之后，我们就可以使用框架内部的`AuthenticationInterceptor`了:

```swift
// 生成授权凭证。用户没有登陆时，可以不生成。
let credential = OAuthCredential(accessToken: "a0",
                                 refreshToken: "r0",
                                 userID: "u0",
                                 expiration: Date(timeIntervalSinceNow: 60 * 60))

// 生成授权中心
let authenticator = OAuthAuthenticator()
// 使用授权中心和凭证（若没有可以不传）配置拦截器
let interceptor = AuthenticationInterceptor(authenticator: authenticator,
                                            credential: credential)

// 将拦截器配置在Session上或在单独的Request中使用
let session = Session()
let urlRequest = URLRequest(url: URL(string: "https://api.example.com/example/user")!)
session.request(urlRequest, interceptor: interceptor)
```

可以看到，使用上面的方式，我们只需关心如何获取`accessToken`和`refreshToken`，以及在`refreshToken`也失效时触发用户重新登录授权。可以说，我们自己的工作少到了极致。自己写的少就意味这 bug 少，特别是刷新 token 这一块，什么时候应该刷新`accessToken`、怎么控制过度刷新这些繁琐的部分我们都无需关心了。

## 如何做到的

知道了怎么做，可能你还会一头雾水。为什么需要定义那两个数据结构？这一部分为你解答。

### AuthenticationCredential

它代表授权凭证，这个协议的定义很简单：

```swift
/// 授权凭证，可以使用它对URLRequest进行授权。
/// 例如：在OAuth2授权体系中，凭证包含accessToken，它可以对一个用户的所有请求进行授权。
/// 通常情况下，该accessToken有效时长为60分钟；在过期前后（一段时间内）可以使用refreshToken对accessToken进行刷新。
public protocol AuthenticationCredential {
    /// 授权凭证是否需要刷新。
    /// 在凭证在即将过期或过期后，应该返回true。
    /// 例如，accessToken的有效期为60分钟，在凭证即将过期的5分钟应该返回true，保证accessToken得到刷新。
    var requiresRefresh: Bool { get }
}
```

该协议只关心这个凭证是否需要刷新。对于不同的授权方式，需要的元信息也不相同，框架无法也无需知道这些细节。

### Authenticator

正因为`AuthenticationCredential`可能五花八门，这里需要一个知道如何使用它的角色。`Authenticator`就来了。该协议的实现细节比较多，我已经写在注释里了。

```swift
/// 授权中心，可以使用凭证（AuthenticationCredential）对URLRequest授权；也可以管理token的刷新。
public protocol Authenticator: AnyObject {
    /// 该授权中心使用的凭证类型
    associatedtype Credential: AuthenticationCredential
    /// 使用凭证对请求进行授权。
    /// 例如：在OAuth2体系中，应该设置请求头 [ "Authorization": "Bearer accessToken" ]
    func apply(_ credential: Credential, to urlRequest: inout URLRequest)

    /// 刷新凭证，并通过completion回调结果。
    /// 在下面两种情况下，会执行刷新：
    /// 1. 适配过程中 - 对应 拦截器的 adapt(_:for:completion:) 方法
    /// 2. 重试过程中 - 对应拦截器的 retry(_:for:dueTo:completion:)方法
    ///
    /// 例如：在OAuth2体系中，应该在该方法中使用refreshToken去刷新accessToken，完成后在回调中返回新的凭证。
    /// 若刷新请求被拒绝（状态码401），refreshToken不应该再使用，此时应该要求用户重新授权。
    func refresh(_ credential: Credential, for session: Session, completion: @escaping (Result<Credential, Error>) -> Void)

    /// 判断URLRequest失败是否因为授权问题。
    /// 若授权服务器不支持对已经生效的凭证进行撤销（也就是说凭证永久有效）应该返回false。否则应该根据具体情况判断。
    /// 例如：在OAuth2体系中， 可以使用状态码401代表授权失败，此时应该返回true。
    /// 注意：上面只是一般情况，你应该根据你所处的系统具体判断。
    func didRequest(_ urlRequest: URLRequest, with response: HTTPURLResponse, failDueToAuthenticationError error: Error) -> Bool

    /// 判断URLRequest是否使用凭证进行了授权。
    /// 若授权服务器不支持对已经生效的凭证进行撤销（也就是说凭证永久有效）应该返回true。否则应该根据具体情况判断。
    /// 例如：在OAuth2体系中，  可以对比`URLRequest中header的授权字段Authorization的值` 和 `Credential中的token`;
    /// 若他们相等，返回true，否则返回false
    func isRequest(_ urlRequest: URLRequest, authenticatedWith credential: Credential) -> Bool
}
```

### AuthenticationInterceptor

为了完成授权流程，该拦截器对请求的适配和重试都进行了实现。

#### Adapter

先上一个适配流程图：
![auth-adapter](http://images-for-blog.oss-cn-beijing.aliyuncs.com/2021/11/29/authadapter.png)

下面是相关代码，我已经加上了详细注释：

```swift
public func adapt(_ urlRequest: URLRequest, for session: Session, completion: @escaping (Result<URLRequest, Error>) -> Void) {
    let adaptResult: AdaptResult = $mutableState.write { mutableState in
        // 适配一个URLRequest时，正在刷新凭证，将此次适配记录下来，延迟执行
        guard !mutableState.isRefreshing else {
            let operation = AdaptOperation(urlRequest: urlRequest, session: session, completion: completion)
            mutableState.adaptOperations.append(operation)
            return .adaptDeferred
        }
        // 没有授权凭证时，报错
        guard let credential = mutableState.credential else {
            let error = AuthenticationError.missingCredential
            return .doNotAdapt(error)
        }
        // 若凭证需要刷新，将此次适配记录下来，延迟执行。并触发刷新操作
        guard !credential.requiresRefresh else {
            let operation = AdaptOperation(urlRequest: urlRequest, session: session, completion: completion)
            mutableState.adaptOperations.append(operation)
            refresh(credential, for: session, insideLock: &mutableState)
            return .adaptDeferred
        }
        // 上面的情况都没有触发，则需要进行适配
        return .adapt(credential)
    }
    switch adaptResult {
    case let .adapt(credential):
        // 使用授权中心进行授权，之后回调
        var authenticatedRequest = urlRequest
        authenticator.apply(credential, to: &authenticatedRequest)
        completion(.success(authenticatedRequest))
    case let .doNotAdapt(adaptError):
        // 出错了就直接回调错误
        completion(.failure(adaptError))
    case .adaptDeferred:
        // 凭证需要刷新或正在刷新， 适配需要延迟到刷新完成后执行
        break
    }
}
```

其中的刷新流程，比就有意思。涉及到刷新窗口的概念。简单讲就是一定的时间范围。在这个范围内，还可以设置一个最大的刷新次数。在正式刷新之前，会判断刷新条件是否满足窗口设定。具体如下：

```swift
/// 判断是否过度刷新
private func isRefreshExcessive(insideLock mutableState: inout MutableState) -> Bool {
    // refreshWindow是判断过度刷新的参考，没有refreshWindow时说明不限制刷新
    guard let refreshWindow = mutableState.refreshWindow else { return false }
    // 计算可刷新的时间点
    let refreshWindowMin = ProcessInfo.processInfo.systemUptime - refreshWindow.interval
    // 统计在可刷新时间点之前的刷新次数
    let refreshAttemptsWithinWindow = mutableState.refreshTimestamps.reduce(into: 0) { attempts, refreshTimestamp in
        guard refreshWindowMin <= refreshTimestamp else { return }
        attempts += 1
    }
    // 若刷新次数 大于等于 配置的最大允许刷新次数，认为过度刷新
    let isRefreshExcessive = refreshAttemptsWithinWindow >= refreshWindow.maximumAttempts

    return isRefreshExcessive
}
```

若上述条件通过，就会执行刷新：

```swift
private func refresh(_ credential: Credential, for session: Session, insideLock mutableState: inout MutableState) {
    // 若过度刷新，直接报错
    guard !isRefreshExcessive(insideLock: &mutableState) else {
        let error = AuthenticationError.excessiveRefresh
        handleRefreshFailure(error, insideLock: &mutableState)
        return
    }
    // 记录刷新时间，设置刷新标志
    mutableState.refreshTimestamps.append(ProcessInfo.processInfo.systemUptime)
    mutableState.isRefreshing = true

    queue.async {
        // 使用授权中心进行刷新。这里就是我们自己实现的授权中心。
        self.authenticator.refresh(credential, for: session) { result in
            self.$mutableState.write { mutableState in
                switch result {
                case let .success(credential):
                    self.handleRefreshSuccess(credential, insideLock: &mutableState)
                case let .failure(error):
                    self.handleRefreshFailure(error, insideLock: &mutableState)
                }
            }
        }
    }
}
```

#### Retrier

还是先看流程图：
![retry](http://images-for-blog.oss-cn-beijing.aliyuncs.com/2021/11/29/retry.png)

这里会判断是否和授权有关，无关的就不会重试。另外，若当前最新凭证没有使用，会进入重试流程。最后的刷新是因为：既然需要授权，也存在凭证，也授权过了，还进入了重试那就说明凭证过期了。下面是具体代码：

```swift
public func retry(_ request: Request, for session: Session, dueTo error: Error, completion: @escaping (RetryResult) -> Void) {
    // 没有原始请求或没有收到服务器的响应，无需重试
    guard let urlRequest = request.request, let response = request.response else {
        completion(.doNotRetry)
        return
    }
    // 不是因为授权原因失败的，无需重试
    guard authenticator.didRequest(urlRequest, with: response, failDueToAuthenticationError: error) else {
        completion(.doNotRetry)
        return
    }
    // 需要授权，却没有凭证的，回调错误
    guard let credential = credential else {
        let error = AuthenticationError.missingCredential
        completion(.doNotRetryWithError(error))
        return
    }
    // 需要授权，但未使用当前凭证，需要重试
    guard authenticator.isRequest(urlRequest, authenticatedWith: credential) else {
        completion(.retry)
        return
    }
    // 需要授权，存在凭证，也授权过了，还进入了重试那就说明凭证过期了，刷新凭证
    $mutableState.write { mutableState in
        mutableState.requestsToRetry.append(completion)
        guard !mutableState.isRefreshing else { return }
        refresh(credential, for: session, insideLock: &mutableState)
    }
}
```

到这里，整个流程也就清晰了。更具体的，可以参考[GitHub](https://github.com/niwaguan/Alamofire/tree/risk)

## 总结

今天我们从具体问题出发，先了解了如何使用`Alamofire`去解决该问题，然后又分析了`AuthenticationInterceptor`的具体实现，它是如何解决该问题的。最后，只能说`Alamofire`真是太细了 😂。
