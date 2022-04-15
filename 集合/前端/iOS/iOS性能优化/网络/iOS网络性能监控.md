# [iOS网络性能监控](https://www.jianshu.com/p/1c34147030d1)



现在的Native App平台化趋势越来越明显，网络层架构也越来越复杂。一个App基本都有多个不同的网络模块。 从简单的业务数据的HTTP/HTTPS(基于NSURLConnection或者NSURLSession),到WebView的WebCore网络层，到基于TCP长连接的推送模块，到各种第三方组件比如统计、日志上报各自的网络层，或者很多app采用基于TCP的私有协议等等，网络层越来越复杂，对Native开发者来说越来越像一个黑盒模块。 Native开发者只能着眼于业务开发，对网络层的异常、性能等等问题一无所知。



# 初识iOS网络层API

让我们剥开网络相关的SDK，一层一层地看每一层做了些什么。

#### AFNetworking

AFNetworking是对NSURLConnection/NSURLSession的封装。增加了如下逻辑

- 封装成NSOperation的形式，提供了resume/cancel等等处理。
- 增加了NSData的文件处理，上传/下载
- 方便处理JSON/XML数据
- 方便处理HTTPS
- 有Reachablity的API

#### NSURLSession/NSURLConnection

NSURLSession/NSURLConnection 都是基于CFNetwork的。

NSURLConnection - CFURLConnection的封装。提供了create,start,cancel,send(同步或者异步)，设置回调，设置runloop等函数。

NSURLSession/NSURLSessionTask - NSCFURLSession/NSCFURLSessionTask等等的封装。

NSURLXXX这一层主要处理了:

- 把CFNetwork的blockhandler封装成delegate的方式
- 处理NSURLProtocol相关的代理
- 处理NSURLCache的缓存相关,是对CFURLCache的封装
- 封装sendAsync/和sendSync方法。
- 把CFURLResponse的statusCode转化成String

#### CFNetwork

CFNetwork 展示了如何把字节流封装成HTTP协议的请求收发。

[图片上传失败...(image-4f1c4b-1535019216015)]

- `CFURLRequest`由用户创建，里面包括URL/header/body这些请求的信息。然后`CFURLRequest`会被转换成`CFHTTPMessage`的格式。
- `CFHTTPMessage`里主要是HTTP协议的定义和转换，把每一个请求request转换成标准的HTTP格式的文本。
- `CFURLConnection` 里主要是处理请求任务,包括pthread线程、CFRunloop,请求队列的管理等等。所以提供了start、cancel等等操作的api。也有操作`CFReadStream`等API
- `CFHost`:负责DNS，在有`CFHostStartInfoResolution`等函数，基于`dns_async_start`和`getaddrinfo_async_start`方法。在iOS8/9基于`getaddrinfo`。主要是同步调用和异步调用的区别。
- `CFURLCache/CFURLCredential/CFHTTPCookie`:处理缓存/证书/cookie相关的逻辑，都有对应的NS类。

主要的数据交换调用基于`CFStream`的API。

#### CFStream

借助`CFSocketStream`,封装`BSD Socket`,和`SecurityTransport`(SSL调用)。

由于`BSD Socket`都是同步调用。所以CFStream这一层主要是Runloop逻辑,锁,dowhile等待等等。
 类似BSD socket一样是数据流输入/输出的API。

`CFStream` 创建时要传入一堆callback，包括open/close,read/write等等。比如`CFSocketStream`，封装了`BSD Socket`的作为callback传入`CFStream`。

`CFSocketStream` 也包括了DNS、SSL连接、Connect握手等等逻辑。

#### BSD Socket

有多组API，包括`connect/shutdown`,`send/recv`,`read/write`,`recvfrom/sendto`,`recvmsg/sendmsg`.
 作为客户端一般不使用`accept/bind`.

`send/recv`和`read/write`的区别在于多了一个`flags`参数。当flag为0时，send等同于write。

对于发送消息。`send`只可用于基于连接的套接字，`sendto` 和 `sendmsg`既可用于无连接的socket，也可用于基于连接的socket。除了socket设置为非阻塞模式，调用将会阻塞直到数据被发送完。

DNS方法:
 `getaddrinfo` 是对 `gethostbyname/gethostbyaddr` 的替代，支持了ipv6，返回一个地址struct链表。在iOS8/9中使用。
 `getaddrinfo_async_start` 在iOS10中使用，支持了异步。

## 监控什么

[iOS-Monitor-Platform](https://github.com/aozhimin/iOS-Monitor-Platform#network) 这篇文章提出了一些监控指标(然而他提供的方法并不能监控到)。

- TCP 建立连接时间
- DNS 时间
- SSL 时间
- 首包时间
- 响应时间
- HTTP 错误率
- 网络错误率
- 流量

APM厂听云提供的一些监控指标:

- TOP5 响应时间最慢主机
- TOP5吞吐率最高主机
- TOP5 DNS时间最慢地域
- TCP建连最慢主机
- 连接次数最多主机

HTTP抓包工具Charles提供的监控指标:

- Request Start Time
- Request End Time
- Response Start Time
- Response End Time
- Duration
- DNS
- Connect
- SSL Handshake
- Latency

当然如果可行的话，想每一个细节都监控到。但是很多数据都有实现成本，本文用最低的成本力求收集尽可能多的指标。

# 具体实现

## HTTP 的监控

HTTP的监控最佳的实践当然就是利用NSURLSession的NSURLSessionTaskMetrics。

[图片上传失败...(image-7b6c8-1535019216015)]

想探究NSURLSessionTaskMetrics的实现，如果反编译CFNetwork的源码，可以看到`-[NSURLSessionTaskMetrics _initWithPerformanceTiming]` 这个方法，说明是来自一个叫`TimingPerformance`的类。
 `TimingPerformance`的初始化方法代码如下，可以看到这里定义了所有NSURLSessionTaskMetrics时间节点需要的key，几乎完全一致。然后初始化时利用`CFAbsoluteTimeGetCurrent`函数来记录初始化的时间。



```cpp
int __ZN17PerformanceTimingC2Ev() {
    rbx = rdi;
    CFObject::CFObject();
    ...
    *(rbx + 0x20) = @"_kCFNTimingDataRedirectStart";
    *(rbx + 0x30) = @"_kCFNTimingDataRedirectEnd";
    *(rbx + 0x40) = @"_kCFNTimingDataFetchStart";
    *(rbx + 0x50) = @"_kCFNTimingDataDomainLookupStart";
    *(rbx + 0x60) = @"_kCFNTimingDataDomainLookupEnd";
    *(rbx + 0x70) = @"_kCFNTimingDataConnectStart";
    *(rbx + 0x80) = @"_kCFNTimingDataConnectEnd";
    *(rbx + 0x90) = @"_kCFNTimingDataSecureConnectionStart";
    *(rbx + 0xa8) = @"_kCFNTimingDataRequestStart";
    *(rbx + 0xb8) = @"_kCFNTimingDataRequestEnd";
    *(rbx + 0xc8) = @"_kCFNTimingDataResponseStart";
    *(rbx + 0xd8) = @"_kCFNTimingDataResponseEnd";
    *(rbx + 0xe8) = @"_kCFNTimingDataRedirectCountW3C";
    *(rbx + 0xf8) = @"_kCFNTimingDataRedirectCount";
    *(rbx + 0x108) = @"_kCFNTimingDataTaskResumed";
    *(rbx + 0x118) = @"_kCFNTimingDataConnectCreate";
    *(rbx + 0x128) = @"_kCFNTimingDataTCPConnected";
    *(rbx + 0x138) = @"_kCFNTimingDataFirstWrite";
    *(rbx + 0x148) = @"_kCFNTimingDataFirstRead";
    *(rbx + 0x158) = @"_kCFNTimingDataConnectionInit";
    *(rbx + 0x168) = @"_kCFNTimingDataConnected";
    ....
    *(rbx + 0x1f0) = @"_kCFNTimingDataTimingDataInit";
    ...
    CFAbsoluteTimeGetCurrent();
    ....
    return rax;
}
```

这个类是怎么使用的呢，可以看到[NSCFURLSessionTask resume]这个方法里:



```cpp
void -[__NSCFURLSessionTask resume](void * self, void * _cmd) {
    rbx = self;
            ...
            __setRecordForKeyInternalPerformanceTiming(@"streamTask-resume");
            r15 = rbx->_performanceTiming;
            if (r15 != 0x0) {
                    PerformanceTiming::Class();
                    xmm0 = intrinsic_movsd(xmm0, *(r15 + 0x110));
                    xmm0 = intrinsic_ucomisd(xmm0, 0x0);
                    if ((xmm0 == 0x0) && (!CPU_FLAGS & P)) {
                            CFAbsoluteTimeGetCurrent();
                            *(r15 + 0x110) = intrinsic_movsd(*(r15 + 0x110), xmm0);
                    }
            }
            __setRecordForKeyInternalPerformanceTiming(@"start-task-resume-to-loader-start-load");
              ...
    return;
}
```

可以看到这里rbx寄存器存储的就是`NSCFURLSessionTask`对象，这个对象有一个成员变量就是_performanceTiming,放在r15这个寄存器里。上面的代码可以看到(0x108)对应的就是`_kCFNTimingDataTaskResumed`这个key，而这里xmm0寄存器是个浮点数存储的寄存器，存储的是`(r15 + 0x110)`，对应应该是，然后判断xmm0是否为空，如果是空的话，就调用`CFAbsoluteTimeGetCurrent`函数获取当前CPU时间，然后再赋给`(r15 + 0x110)`，对应的应该就是`_kCFNTimingDataTaskResumed`这个key对应的value。

至于`__setRecordForKeyInternalPerformanceTiming` 这个函数，可以看到它的key并不存在于PerformanceTiming对象初始化的时候，它应该是`InternalPerformanceTiming`，这是个不同的类，可能是`PerformanceTiming`的子类。他的key是不同的，判断是这个库内部使用的，并没有作为NSURLSessionTaskMetrics传递出去。

发现`__ZN17PerformanceTiming32fillW3NavigationTimingAWDMetricsEP27PerformanceTimingAWDMetrics`,`__ZN17PerformanceTiming30fillStreamTaskTimingAWDMetricsEP26StreamTaskTimingAWDMetrics`这两个函数，说明`PerformanceTiming`和`W3NavigationTiming`，以及`StreamTaskTiming`这几个东西的`AWDMetrics`是可以互相转化的。`W3NavigationTiming`很容易想到是用于WebView的Timing的API。

NSURLSessionTaskMetrics的优点是苹果帮咱实现了，但是有很严重的缺点是只能适用于iOS10以后的NSURLSession。~~NSURLConnection是用不了的。iOS10以下也是用不了的。~~ (见后文重大发现)

##### 其它方案的分析

对于iOS10以下的NSURLSession以及NSURLConnection,想要打点统计时间点挺困难的，主要困难点在不同SDK的API调用不同,比如iOS8和9的DNS,可以hook到`getaddrinfo`函数,iOS10有时可以hook到`getaddrinfo_async_start`函数，但是对于iOS11，我尝试了各种跟DNS相关的函数，完全hook不到。反编译CFNetwok出来的跟DNS相关的函数，也都没有被`NSURLSession/NSURLConnection`调用。 SSL的情况也非常类似，目前只知道iOS8/9会通过SecurityTransport的`SSLHandshake/SSLRead/SSLWrite`等函数,但是iOS10以上就完全懵逼。这些尝试只能宣告失败，告一段落了。

有的文章认为是iOS10之后系统屏蔽了某些BSD Socket函数的hook,比如connect/read/write 等等。 据我观察并不是这样,BSD socket还是能够hook到，只是大部分情况下不调用这些API了，少数情况还是有使用的。 如果真的被屏蔽了，应该是完全hook不到的。

有些文章写了说监控HTTP，可以采用`NSURLProtocol`拦截请求的方式(比如听云)。我都是持怀疑态度的，因为监控性能，如果没有`DNS/SSL`相关的监控就失去了大部分意义，而监控request/response的就不需要用hook这种方式了(完全可以在自己封装的网络层部分实现)。  而针对于NSURL相关API的hook是统计不到DNS/SSL的，因为它们不在这一层实现。

也有些文章说hook CFStream的方式(比如网易APM)。 但是如果看到CFStream的实现就知道CFStream是对BSD Socket的封装，Open/Close/read/Write 如果看CFSocketStream的源码，这些API都还是BSD Socket实现的，就是说hook CFStream 和 BSD socket没有很大的区别。 还是没有hook到DNS/SSL的点上。

## WebView 的监控

WebView的监控是相对简单的，主要是Timing API。

[图片上传失败...(image-56e48e-1535019216015)]

好处是兼容性很好，目前UIWebView和WKWebView都支持，iOS9以上都支持。因为是浏览器的API。

WebCore里，跟这个timing相关的API主要是PerformanceTiming类:



```cpp
class PerformanceTiming : public RefCounted<PerformanceTiming>, public DOMWindowProperty {
public:
    static Ref<PerformanceTiming> create(Frame* frame) { return adoptRef(*new PerformanceTiming(frame)); }

    unsigned long long navigationStart() const;
    unsigned long long unloadEventStart() const;
    unsigned long long unloadEventEnd() const;
    unsigned long long redirectStart() const;
    unsigned long long redirectEnd() const;
    unsigned long long fetchStart() const;
    //...省略部分函数
    unsigned long long domContentLoadedEventStart() const;
    unsigned long long domContentLoadedEventEnd() const;
    unsigned long long domComplete() const;
    unsigned long long loadEventStart() const;
    unsigned long long loadEventEnd() const;

private:
    explicit PerformanceTiming(Frame*);
    const DocumentTiming* documentTiming() const;
    DocumentLoader* documentLoader() const;
    LoadTiming* loadTiming() const;
};

} // namespace WebCore
```

头文件里有一堆getter函数的定义，同时初始化方法只有一个，入参是单一的`Frame`对象，说明一个Frame对象就能够提供到这些所有的参数。



```cpp
unsigned long long PerformanceTiming::requestStart() const
{
    DocumentLoader* loader = documentLoader();
    if (!loader)
        return connectEnd();

    const NetworkLoadMetrics& timing = loader->response().deprecatedNetworkLoadMetrics();
    ASSERT(timing.requestStart >= 0_ms);
    return resourceLoadTimeRelativeToFetchStart(timing.requestStart);
}
unsigned long long PerformanceTiming::domInteractive() const
{
   const DocumentTiming* timing = documentTiming();
   if (!timing)
       return 0;
   return monotonicTimeToIntegerMilliseconds(timing->domInteractive);
}
unsigned long long PerformanceTiming::loadEventStart() const
{
    LoadTiming* timing = loadTiming();
    if (!timing)
        return 0;
    return monotonicTimeToIntegerMilliseconds(timing->loadEventStart());
}
```

再看cpp文件就知道，PerformanceTiming是对Frame类中已经统计好的参数的一个封装，内部并没有逻辑。数据其实就是来自于`NetworkLoadMetrics`、`DocumentTiming` 和 `LoadTiming` 三部分。也很容易理解就是分别对应网络请求相关的性能统计、对应DOM加载相关的和WebView加载相关的性能统计。

有一个细节就是NetworkLoadMetrics里有0ms的判断，保证`NetworkLoadMetrics`返回的相关数据大于0。而`DocumentTiming` 和 `LoadTiming`返回的数据为空时就是0。实际上使用这一系列数据时确实会出现一部分参数为0的情况，而且跟调用PerformanceTiming的接口有关。

WebCore的类的架构如下图。
 [图片上传失败...(image-449013-1535019216015)]
 那WebView里，网络是怎么一层层调用的呢？ 追踪WKWebView的`loadRequest:`方法，调用栈应该是这样的:



```cpp
- (WKNavigation *)loadRequest:(NSURLRequest *)request
void WebPage::loadRequest(const LoadParameters& loadParameters)
void UserInputBridge::loadRequest(FrameLoadRequest&& request, InputSource)
void FrameLoader::load(FrameLoadRequest&& request)
void FrameLoader::load(DocumentLoader* newDocumentLoader)
void FrameLoader::loadWithDocumentLoader(DocumentLoader* loader, FrameLoadType type, FormState* formState, AllowNavigationToInvalidURL allowNavigationToInvalidURL)
void FrameLoader::continueLoadAfterNavigationPolicy(const ResourceRequest& request, FormState* formState, bool shouldContinue, AllowNavigationToInvalidURL allowNavigationToInvalidURL)
void DocumentLoader::startLoadingMainResource()
void ResourceLoader::start()
void ResourceHandle::createNSURLConnection(id delegate, bool shouldUseCredentialStorage, bool shouldContentSniff, SchedulingBehavior, NSDictionary *connectionProperties);
```

ResourceLoader是资源加载，而真正操作网络请求的类在ResourceHandle。看代码就发现WebCore的网络层在iOS上也是基于NSURLConnection的。可以通过AOP的方式hook到某个位置，然后使用NSURLConnection的API进行操作。

有一处比较有意思，WebCore中实现了一个NSURLSession，叫WebCoreNSURLSession。这个类似乎只在MediaPlayer里面使用。相同的是他们也有类似的API，比如`dataTaskWithRequest:`等等，但是内部实现不一样，WebCoreNSURLSession也是基于ResourceLoader的子类。而NSURLSession是基于CFNetwork。

使用Timing系列的API也有需要注意的细节。WebCore内核在iOS上和在Mac的Safari上是不一样的。iOS10以后的WKWebView才实现`.toJSON()`。如果是UIWebView，或者是iOS10以下的WKWebView，需要先执行一段js脚本，方便我们把js对象转换为json。



```objectivec
NSString *funcStr = @"function flatten(obj) {"
        "var ret = {}; "
        "for (var i in obj) { "
        "ret[i] = obj[i];"
        "}"
        "return ret;}";
[webView stringByEvaluatingJavaScriptFromString:funcStr];
```

## TCP的监控

一般App的网络层长连接，会基于TCP实现自定义的协议或者使用Websocket。有的app会基于`BSD Socket`封装(比如微信的`mars`)。有的会先利用一些开源的框架比如 `CocoaAsyncSocket` 或者 `SocketRocket`，然后再进行封装。

#### CocoaAsyncSocket

CocoaAsyncSocket是基于`BSD Socket`,`CFStream`,`SecurityTransport`的封装，封装成TCP/UDP协议。这几个API的共同之处在于都是数据流读写的形式。`BSD Socket`主要是同步阻塞，而`CFStream`是异步的。

既然是数据流读写，所以CocoaAsyncSocket肯定是包括数据流的处理和转换了。主要是缓冲区，ReadBuffer/WriteBuffer,判断读取的结尾CRLF, 读取的长度length和读取的超时机制等等。
 CocoaAsyncSocket也封装了DNS、ipv4和ipv6、SSL等等逻辑。

#### SocketRocket

是基于`NSStream` 的封装，不同于CocoaAsyncSocket的传输层协议,  支持HTTP/WebSocket的应用层协议，定义了header的字段等等。
 由于是基于数据流读写的，所以也包括readBuffer/WriteBuffer等数据处理逻辑。
 也包括Runloop，线程等异步处理逻辑和阻塞同步逻辑。
 还实现了PingPong这样的，跟服务端配合的保活逻辑。

所以TCP的监控可以hook `BSD Socket` 的API，包括 `connect/disconnect/read/write`等等调用，如果是同步调用，所以可以在执行函数前后埋点计算时间。
 也需要hook DNS方面的API，比如 `gethostbyname/getaddrinfo`等同步调用的以及`getaddrinfo_async_start`等等异步调用的API。
 也可以hook SSL 方面的API，比如  `SSLHandshake/SSLRead/SSLWrite`，实现对SSL连接的监测。

## 剧情反转，重大发现 (这一段是后来加的)

HTTP的性能监控因为NSURLSessionTaskMetrics的兼容性问题似乎已经穷途末路了，但是在写第二部分WebView的时候突然有了一个巨大的发现，这个发现来自于在看WebCore的源码的时候发现了一些神奇的东西。



```objectivec
#if !HAVE(TIMINGDATAOPTIONS)
void setCollectsTimingData()
{
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        [NSURLConnection _setCollectsTimingData:YES];
        ...
    });
}
#endif
```

这是说NSURLConnection本身有一套`TimingData`的收集API，只是没有暴露给开发者而已，但是WebCore里一直在用...苹果你为啥这么小气？！
 然后就很轻易地在runtime header里找到了NSURLConnection的`_setCollectsTimingData:` API，还有`_timingData`的API。
 这货iOS8以后都是支持的，iOS8之前也许也支持了。

那么NSURLSession呢，是不是也类似？果然。在iOS9之前，也只需要设置`_setCollectsTimingData:`就好了。
 搜了一下google和github，我应该是第一个发现这个私有API的人...

所以很神奇地，很轻易地，就实现了NSURLConnection和NSURLSession全套的支持....

## 总结

我们几乎可以用很少的代码实现HTTP/WebView/TCP跨框架的大部分网络性能数据收集。如果把兼容性整理成一张表的话可以看到我们几乎支持了大部分的场景。

| iOS SDK | NSURLConnetion | NSURLSession | UIWebView | WKWebView | TCP  |
| ------- | -------------- | ------------ | --------- | --------- | ---- |
| 8.4     | YES            | YES          | via TCP   | via TCP   | YES  |
| 9.3     | YES            | YES          | YES       | YES       | YES  |
| 10.3    | YES            | YES          | YES       | YES       | YES  |
| 11.3    | YES            | YES          | YES       | YES       | YES  |

[NetworkTracker](https://github.com/lilidan/NetworkTracker) 是我封装的一部分代码。并将监控结果简单地画了个图表。还是比较直观的。

![img](https:////upload-images.jianshu.io/upload_images/611240-eb568aaf8b54e858.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)



