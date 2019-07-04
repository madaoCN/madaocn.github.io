---
layout: post
title: iOS使用NSURLProtocol来Hook拦截WKWebview请求并回放的一种姿(ti)势(wei)
date: 2017-11-28
Author: madaoCN
categories: 
tags: [源码分享]
comments: true
---
> 有些时候我们难免需要和 __WKWebView__ 做一些交互，虽然__WKWebView__性能高，但是坑还是不少的

例如：我们在__UIWebview__ ,可以通过如下方式获取js上下文，但是在__WKWebView__是会报错的

>
```objc
let context = webView.valueForKeyPath("documentView.webView.mainFrame.javaScriptContext") as! JSContext
context.evaluateScript(theScript)
```
> 公司服务端自定义了一些模式，例如：custom://action?param=1 来对客户端做些控制，那么我们就需要对自定义的模式进行拦截和请求，但是下文不仅会hook拦截自定义模式，还会拦截`https`和`http`的请求

额外的玩意儿：

其实 __WKWebView__ 自带了一些和 __JS__ 交互的接口

* __WKUserContentController__ 和 __WKUserScript__
通过`- (void)addUserScript:(WKUserScript *)userScript;`接口对 __JS__ 做控制
__JS__ 通过` window.webkit.messageHandlers.<name>.postMessage(<messageBody>)`来给原生发送消息
  然后原生通过以下方法来响应请求
```swift
- (void)addScriptMessageHandler:(id <WKScriptMessageHandler>)scriptMessageHandler name:(NSString *)name;
```
* __evaluateJavaScript:completionHandler:__ 方法
__WKWebview__ 自带了异步调用 js代码的接口
```swift
- (void)evaluateJavaScript:(NSString *)javaScriptString completionHandler:(void (^ _Nullable)(_Nullable id, NSError * _Nullable error))completionHandler;
```
然后，通过 __WKScriptMessageHandler__ 协议方法
```swift
- (void)userContentController:(WKUserContentController *)userContentController didReceiveScriptMessage:(WKScriptMessage *)message;
```
来处理 __JS__ 给过来的请求

还有一些原生__JavaScriptCore__ 和 __JS__ 交互的一些知识请看本人另一篇博客 [JavaScriptCore与JS交互笔记](http://www.jianshu.com/p/3332185e3fdf)

> 扯了这么多，进入正题吧

个人觉得通过拦截自定义模式的方式来处理请求会灵活一些，接下来的内容要解决几个问题
> * __自定义拦截请求协议（https，http，customProtocol等等）__
> *  __对拦截的 __WKWebView__ 请求做处理，不仅接管请求还要将请求结果返还给__WKWebView__.__

##### 那么，开始吧

在 __UIWebview__ 时期，使用 __NSURLProtocol__ 可以拦截到网络请求, 但是
> WKWebView 在独立于 App Process 进程之外的进程中执行网络请求，请求数据不经过主进程，因此，在__WKWebView__ 上直接使用 __NSURLProtocol__ 无法拦截请求

但是 接下来我们还是要用 __NSURLProtocol__ 来拦截，但是需要一些 tirick

我们可以使用私有类 __WKBrowsingContextController__   通过 __registerSchemeForCustomProtocol__ 方法向 __WebProcessPool__ 注册全局自定义 scheme 来达到我们的目的

在 __application:didFinishLaunchingWithOptions__ 方法中执行如下语句，对需要拦截的协议进行注册
```swift
- (void)registerClass
{
    // 防止苹果静态检查 将 WKBrowsingContextController 拆分，然后再拼凑起来
    NSArray *privateStrArr = @[@"Controller", @"Context", @"Browsing", @"K", @"W"];
    NSString *className =  [[[privateStrArr reverseObjectEnumerator] allObjects] componentsJoinedByString:@""];
    Class cls = NSClassFromString(className);
    SEL sel = NSSelectorFromString(@"registerSchemeForCustomProtocol:");
    
    if (cls && sel) {
        if ([(id)cls respondsToSelector:sel]) {
            // 注册自定义协议
            // [(id)cls performSelector:sel withObject:@"CustomProtocol"];
            // 注册http协议
            [(id)cls performSelector:sel withObject:HttpProtocolKey];
            // 注册https协议
            [(id)cls performSelector:sel withObject:HttpsProtocolKey];
        }
    }
   // SechemaURLProtocol 自定义类 继承于 NSURLProtocol
    [NSURLProtocol registerClass:[SechemaURLProtocol class]];
}
```

上述用到了一个继承 __NSURLProtocol__ 的自定义类 __SechemaURLProtocol__

我们主要需要复写如下几个方法
```swift

// 判断请求是否进入自定义的NSURLProtocol加载器
+ (BOOL)canInitWithRequest:(NSURLRequest *)request;

// 重新设置NSURLRequest的信息, 这方法里面我们可以对请求做些自定义操作，如添加统一的请求头等
+ (NSURLRequest *) canonicalRequestForRequest:(NSURLRequest *)request;

// 被拦截的请求开始执行的地方
- (void)startLoading;

// 结束加载URL请求
- (void)stopLoading;
```

完整的代码
```swift
+ (BOOL)canInitWithRequest:(NSURLRequest *)request
{
    NSString *scheme = [[request URL] scheme];
    if ([scheme caseInsensitiveCompare:HttpProtocolKey] == NSOrderedSame ||
        [scheme caseInsensitiveCompare:HttpsProtocolKey] == NSOrderedSame)
    {
        //看看是否已经处理过了，防止无限循环
        if ([NSURLProtocol propertyForKey:kURLProtocolHandledKey inRequest:request]) {
            return NO;
        }
    }
    
    return YES;
}

+ (NSURLRequest *) canonicalRequestForRequest:(NSURLRequest *)request {
    
    NSMutableURLRequest *mutableReqeust = [request mutableCopy];
    return mutableReqeust;
}

// 判重
+ (BOOL)requestIsCacheEquivalent:(NSURLRequest *)a toRequest:(NSURLRequest *)b
{
    return [super requestIsCacheEquivalent:a toRequest:b];
}

- (void)startLoading
{
    NSMutableURLRequest *mutableReqeust = [[self request] mutableCopy];
    // 标示改request已经处理过了，防止无限循环
    [NSURLProtocol setProperty:@YES forKey:kURLProtocolHandledKey inRequest:mutableReqeust];
}

- (void)stopLoading
{
}
```
现在我们已经解决了第一个问题
> * __自定义拦截请求协议（https，http，customProtocol等等）__

但是，如果我们 hook 了 __WKWebview__ 的 __http__ 或者 __https__请求，就等于我们接管了该请求，我们需要手动控制它的请求声明周期，并在适当的时候返还回放给  __WKWebview__， 否则 __WKWebview__ 将始终无法显示被hook请求的加载结果

那么，接下来我们使用 __NSURLSession__ 来发送和管理请求，__PS__ 笔者尝试过使用 __NSURLConnection__ 但是没有请求成功

在这之前, __NSURLProtocol__ 有个遵循了  __NSURLProtocolClient__ 协议的 __client__ 属性
```swift
/*!
    @abstract Returns the NSURLProtocolClient of the receiver. 
    @result The NSURLProtocolClient of the receiver.  
*/
@property (nullable, readonly, retain) id <NSURLProtocolClient> client;
```
我们需要通过这个 __client__ 来向 __WKWebview__ 沟通消息

__NSURLProtocolClient__ 协议方法
```swift
@protocol NSURLProtocolClient <NSObject>

// 重定向请求
- (void)URLProtocol:(NSURLProtocol *)protocol wasRedirectedToRequest:(NSURLRequest *)request redirectResponse:(NSURLResponse *)redirectResponse;

// 响应缓存是否可用
- (void)URLProtocol:(NSURLProtocol *)protocol cachedResponseIsValid:(NSCachedURLResponse *)cachedResponse;

// 已经接收到Response响应
- (void)URLProtocol:(NSURLProtocol *)protocol didReceiveResponse:(NSURLResponse *)response cacheStoragePolicy:(NSURLCacheStoragePolicy)policy;

// 成功加载数据
- (void)URLProtocol:(NSURLProtocol *)protocol didLoadData:(NSData *)data;

// 请求成功 已经借宿加载
- (void)URLProtocolDidFinishLoading:(NSURLProtocol *)protocol;

// 请求加载失败
- (void)URLProtocol:(NSURLProtocol *)protocol didFailWithError:(NSError *)error;

@end
```

我们需要在  __NSURLSessionDelegate__ 代理方法中合适的位置让__client__ 调用 __NSURLProtocolClient__ 协议方法

我们在 __- (void)startLoading__ 中发送请求
```swift

NSURLSessionConfiguration *configure = [NSURLSessionConfiguration defaultSessionConfiguration];
self.session  = [NSURLSession sessionWithConfiguration:configure delegate:self delegateQueue:self.queue];
self.task = [self.session dataTaskWithRequest:mutableReqeust];
[self.task resume];
```

__NSURLSessionDelegate__ 请求代理方法

```swift
- (void)URLSession:(NSURLSession *)session task:(NSURLSessionTask *)task didCompleteWithError:(NSError *)error
{
    if (error != nil) {
        [self.client URLProtocol:self didFailWithError:error];
    }else
    {
        [self.client URLProtocolDidFinishLoading:self];
    }
}

- (void)URLSession:(NSURLSession *)session dataTask:(NSURLSessionDataTask *)dataTask
didReceiveResponse:(NSURLResponse *)response
completionHandler:(void (^)(NSURLSessionResponseDisposition disposition))completionHandler
{
    [self.client URLProtocol:self didReceiveResponse:response cacheStoragePolicy:NSURLCacheStorageNotAllowed];
    
    completionHandler(NSURLSessionResponseAllow);
}

- (void)URLSession:(NSURLSession *)session dataTask:(NSURLSessionDataTask *)dataTask didReceiveData:(NSData *)data
{
    [self.client URLProtocol:self didLoadData:data];
}

- (void)URLSession:(NSURLSession *)session dataTask:(NSURLSessionDataTask *)dataTask willCacheResponse:(NSCachedURLResponse *)proposedResponse completionHandler:(void (^)(NSCachedURLResponse * _Nullable))completionHandler
{
    completionHandler(proposedResponse);
}

//TODO: 重定向
- (void)URLSession:(NSURLSession *)session task:(NSURLSessionTask *)task willPerformHTTPRedirection:(NSHTTPURLResponse *)response newRequest:(NSURLRequest *)newRequest completionHandler:(void (^)(NSURLRequest *))completionHandler
{
    NSMutableURLRequest*    redirectRequest;
    redirectRequest = [newRequest mutableCopy];
    [[self class] removePropertyForKey:kURLProtocolHandledKey inRequest:redirectRequest];
    [[self client] URLProtocol:self wasRedirectedToRequest:redirectRequest redirectResponse:response];
    
    [self.task cancel];
    [[self client] URLProtocol:self didFailWithError:[NSError errorWithDomain:NSCocoaErrorDomain code:NSUserCancelledError userInfo:nil]];
}
```

到此，我们已经解决了第二个问题
> __对拦截的 __WKWebView__ 请求做处理，不仅接管请求还要将请求结果返还给__WKWebView__.__

笔者，将以上代码封装成了一个简单的Demo，实现了Hook __WKWebView__ 的请求，并显示在界面最下层的Label中

![Untitled.gif](http://upload-images.jianshu.io/upload_images/1749699-21b2be820d2eafe1.gif?imageMogr2/auto-orient/strip%7CimageView2/2/w/640)

[DEMO   Github地址：https://github.com/madaoCN/WKWebViewHook](https://github.com/madaoCN/WKWebViewHook)

有路过的同学点个喜欢再走呗