# [Flutter之禅 内存优化篇](https://www.dengtar.com/18406.html)



## 前言

性能稳定性是App的生命，Flutter带了很多创新与机遇，然而团队在享受Flutter带来的收益同时也迎接了很多新事物带来的挑战。

本文就内存优化过程中一些实践经验跟大家做一个分享。

## Flutter 上线之后

闲鱼使用一套混合栈管理的方案将Flutter嵌入到现有的App中。在产品体验上我们取得了优于Native的体验。主要得益于Flutter的在跨平台渲染方面的优势，部分原因则是因为我们用Dart语言重新实现的页面抛弃了很多历史的包袱轻装上阵。

上线之后各方面技术指标，都达到甚至超出了部分预期。而我们最为担心的一些稳定性指标，比如crash也在稳定的范围之内。但是在一段时间后我们发现由于内存过高而被系统杀死的abort率数据有比较明显的异常。性能稳定性问题是非常关键的，于是我们火速开展了问题排查。

## 问题定位与排查

显然问题出在了过大的内存消耗上。内存消耗在App中构成比较复杂，如何在复杂的业务中去定位到罪魁祸首呢？稍加观察，我们确定Flutter问题相对比价明显。工欲善其事必先利其器，需要更好地定位内存的问题，善用已经的工具是非常有帮助的。好在我们在Native层和Dart层都有足够多的性能分析工具进行使用。

### 工具分析

这里简单介绍我们如何使用的工具去观察手机数据以便于分析问题。需要注意的是，本文的重点不是工具的使用方法介绍，所以只是简单列举部分使用到的常见工具。

#### Xcode Instruments

Instruments是iOS内存排查的利器，可以比较便捷地观察实时内存使用情况，自然不必多说。

#### Xcode MemGraph + VMMap

XCode 8之后推出的MEMGraph是Xcode的内存调试利器，可以看到实时的可视化的内存。更为方便的是，你可以将MemGraph导出，配合命令行工具更好的得到结构化的信息。

#### Dart Observatory

这是Dart语言官方的调试工具，里面也包含了类似于Xcode的Instruments的工具。在Debug模式下Dart VM启动以后会在特定的端口接受调试请求。官方文档

#### 观察结果

在整个过程中我进行了大量的观察，这里分享一部分典型的数据表现。

通过Xcode Instruments排查的话，我们观察到CG Raster Data这个数据有些高。这个Raster Data呢其实是图片光栅化的时候的内存消耗。

我们将App内存异常的场景的MemGraph导出来，对其执行VMMap指令得出的结果:

```
vmmap --summary Runner[40957].memgraphvmmap Runner[40957].memgraph | grep 'IOKit' 
```

![img](https://www.dengtar.com/res/201903/01/1549936680_0_956.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/640/format/webp)vmmap Summary![img](https://www.dengtar.com/res/201903/01/1549936680_1_138.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/640/format/webp)vmmap address

我们主要关注resident和dirty的内存。发现IOKit占用了大量的内存。

结合Xcode Raster Data还有IOKit的大量内存消耗，我们开始怀疑问题是图内存泄漏导致的。经过进一步通过Dart Observatory观察Dart Image对象的内存情况。


![img](https://www.dengtar.com/res/201903/01/1549936680_2_367.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/640/format/webp)Dart image instance

观察结果显示，在内存较高的场景下在Dart层的确同时存在了较多Image(如图中270)的对象。现在基本可以确定内存问题跟Dart层的图片有很大的关系。

这个结果，我估计很多人都已经想到了，App有明显的内存问题很有可能就是跟多媒体资源有关系。通过工具得出的准确数据线索，我们得到一个大致的方向去深入研究。

## 诡异的Dart图片数量爆炸

### 图片对象泄漏？

前面我们用工具观察到Dart层的Image对象数量过多直接导致了非常大的内存压力，我们起初怀疑存在图片的内存泄漏。但是我们在经过进一步确认以后发现图片其实并没有真正的泄漏。

Dart语言采用垃圾回收机制（Garbage Collection 下面开始简称GC）来管理分配的内存，VM层面的垃圾回收应该大多数情况下是可信的。但是从实际观察来看，图片数量的爆炸造成的较大的内存峰值直观感觉上GC来得有些不及时。在Debug模式下我们使用Dart Observatory手动触发GC，最终这些图片对象在没有引用的情况下最终还是会被回收。

至此，我们基本可以确认，图片对象不存在泄漏。那是什么导致了GC的反应迟钝呢，难道是Dart语言本身的问题吗？

### Garbage Collection 不及时？

为此我需要了解一下Dart内存管理机制垃圾回收的实现，关于详细的内存问题我团队的 @匠修 同学已经发过一篇相关文章可以参考：内存文章

我这里不详细讨论Dart垃圾回收实现细节，只聊一聊Flutter与Dart相关的一些内容。
[图片上传失败...(image-8cfc73-1539229685285)]
关于Flutter我需要首先明确几个概念：

1. Framework（Dart）（跟iOS平台连接的库Flutter.framework要区别开）特指由Dart编写的Flutter相关代码。
2. Dart VM执行Dart代码的Dart语言相关库，它是以C实现的Dart SDk形式提供的。对外主要暴露了C接口Dart Api。里面主要包含了Dart的编译器，运行时等等。
3. FLutter Engine C++实现的Flutter驱动引擎。他主要负责跨平台的绘制实现，包含Skia渲染引擎的接入；Dart语言的集成；以及跟Native层的适配和Embeder相关的一些代码。简单理解，iOS平台上面Flutter.framework, Android平台上的Flutter.jar便是引擎代码构建后的产物。

在Dart代码里面对于GC是没有感知的。

对于Dart SDK也就是Dart语言我们可以做的很有限，因为Dart语言本身是一种标准，如果Dart真的有问题我们需要和Dart维护团队协作推进问题的解决。Dart语言设计的时候初衷也是希望GC对于使用者是透明的，我们不应该依赖GC实现的具体算法和策略。不过我们还是需要通过Dart SDK的源码去理解GC的大致情况。

既然我们前面已经确认并非内存泄漏，所以我们在对GC延迟的问题的调查主要放在Flutter Engine以及Dart CG入口上。

#### Flutter与Dart Garbage Collection

既然感觉GC不及时，先撇开消耗，我们至少可以尝试多触发几次GC来减轻内存峰值压力。但是我在仔细查阅dart_api.h(`/src/third_party/dart/runtime/include/dart_api.h` )接口文件后，但是并没有找到显式提供触发GC的接口。

但是找到了如下这个方法`Dart_NotifyIdle`：

```
/*** Notifies the VM that the embedder expects to be idle until |deadline|. The VM* may use this time to perform garbage collection or other tasks to avoid* delays during execution of Dart code in the future.** |deadline| is measured in microseconds against the system's monotonic time.* This clock can be accessed via Dart_TimelineGetMicros().** Requires there to be a current isolate.*/ DART_EXPORT void Dart_NotifyIdle(int64_t deadline); 
```

这个接口意思是我们可以在空闲的时候显式地通知Dart，你接下来可以利用这些时间（dealine之前）去做GC。注意，这里的GC不保证会马上执行，可以理解我们请求Dart去做GC，具体做不做还是取决于Dart本身的策略。

另外，我还找到一个方法叫做`Dart_NotifyLowMemory`:

```
/*** Notifies the VM that the system is running low on memory.** Does not require a current isolate. Only valid after calling Dart_Initialize.*/ DART_EXPORT void Dart_NotifyLowMemory(); 
```

不过这个Dart_NotifyLowMemory方法其实跟GC没有太大关系，它其实是在低内存的情况下把多余的isolate去终止掉。你可以简单理解，把一些不是必须的线程给清理掉。

在研究Flutter Engine代码后你会发现，Flutter Engine其实就是通过Dart_NotifyIdle去跟Dart层进行GC方面的协作的。我们可以在Flutter Engine源码animator.cc看到以下代码：

```
 //Animator负责刷新和通知帧的绘制 if (!frame_scheduled_) { // We don't have another frame pending, so we're waiting on user input // or I/O. Allow the Dart VM 100 ms. delegate_.OnAnimatorNotifyIdle(*this, dart_frame_deadline_ + 100000); } //delegate 最终会调用到这里 bool RuntimeController::NotifyIdle(int64_t deadline) { if (!root_isolate_) { return false; }tonic::DartState::Scope scope(root_isolate_.get()); //Dart api接口 Dart_NotifyIdle(deadline); return true; }
```

这里的逻辑比较直观：如果当前没有帧渲染的任务时候就通过`NotifyIdle`告诉Dart层可以进行GC操作了。注意，这里并不是说只有在这种情况下Dart才回去做GC，Flutter只是通过这种方式尽可能利用空闲去做GC，配合Dart以更合理的时间去做GC。

看到这里，我们有足够的理由去尝试一下这个接口，于是我们在一些内存压力比较大的场景进行了手动请求GC的操作。线上的Abort虽然有明显好转，但是内存峰值并没有因此得到改善。我们需要进一步找到根本原因。

### 图片数量爆炸的真相

为了确定图片大量囤积释放不及时的问题，我们需要跟踪Flutter图片从初始化到销毁的整个流程。

我们从Dart层开始去追寻Image对象的生命周期，我们可以看到Flutter里面所以的图片都是经过ImageProvider来获取的，ImageProvider在获取图片的时候会调用一个Resolve接口，而这个接口会首先查询ImageCache去读取图片，如果不存在缓存就new Image的实例出来。

关键代码：

```
ImageStream resolve(ImageConfiguration configuration) { assert(configuration != null); final ImageStream stream = new ImageStream(); T obtainedKey; obtainKey(configuration).then<void>((T key) { obtainedKey = key; stream.setCompleter(PaintingBinding.instance.imageCache.putIfAbsent(key, () => load(key))); }).catchError( (dynamic exception, StackTrace stack) async { FlutterError.reportError(new FlutterErrorDetails( exception: exception, stack: stack, library: 'services library', context: 'while resolving an image', silent: true, // could be a network error or whatnot informationCollector: (StringBuffer information) { information.writeln('Image provider: $this'); information.writeln('Image configuration: $configuration'); if (obtainedKey != null) information.writeln('Image key: $obtainedKey'); } )); return null; } ); return stream; } 
```

大致的逻辑

1. Resolve 请求获取图片.
2. 查询是否存在于ImageCache.Yes->3 NO->4
3. 返回已经存在的图片对象
4. 生成新的Image对象并开始加载
   看起来没有特别复杂的逻辑，不过这里我要提一下Flutter ImageCache的实现。

#### Flutter ImageCache

Flutter ImageCache最初的版本其实非常简单，用Map实现的基于LRU算法缓存。这个算法和实现没有什么问题，但是要注意的是ImageCache缓存的是ImageStream对象，也就是缓存的是一个异步加载的图片的对象。而且缓存没有对占用内存总量做限制，而是采用默认最大限制1000个对象（Flutter在0.5.6 beta中加入了对内存大小限制的逻辑）。缓存异步加载对象的一个问题是，在图片加载解码完成之前，无法知道到底将要消耗多少内存，至少在Flutter这个Cache实现中没有处理这个问题。具体的实现感兴趣的朋友可以阅读ImageCache.dart源码。

其实Flutter本身提供了定制化Cache的能力，所以优化ImageCache的第一步就是要根据机型的物理内存去做缓存大小的适配，设置ImageCache的合理限制。关于ImageCache的问题，可以参考官方文档和这个issue，我这里不展开去聊了。

#### Flutter Image生命周期

回到我们的Image对象跟踪，很明显，在缓存没有命中的情况下会有新的Image产生。继续深入代码会发现Image对象是由这段代码产生的：

```
 Future<Codec> instantiateImageCodec(Uint8List list) { return _futurize( (_Callback<Codec> callback) => _instantiateImageCodec(list, callback, null) ); }String _instantiateImageCodec(Uint8List list, _Callback<Codec> callback, _ImageInfo imageInfo) native 'instantiateImageCodec'; 
```

这里有个native关键字，这是Dart调用C代码的能力，我们查看具体的源码可以发现这个最终初始化的是一个C++的codec对象。具体的代码在Flutter Engine codec.cc。它大致的过程就是先在IO线程中启动了一个解码任务，在IO完成之后再把最终的图片对象发回UI线程。关于Flutter线程的详细介绍，我在另外一篇文章中已经有介绍，这里附上链接给有兴趣的朋友。深入理解Flutter Engine线程模型。经过来这些代码和线程分析，我们得到大致的流程图：

![img](https://www.dengtar.com/res/201903/01/1549936680_3_353.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/640/format/webp)图片爆炸流程图

也就是说，解码任务在IO线程进行，IO任务队列里面都是C++ lambda表达式，持有了实际的解码对象，也就持有了内存资源。当IO线程任务过多的时候，会有很多IO任务在等待执行，这些内存资源也被闭包所持有而等待释放。这就是为什么直观上会有内存释放不及时而造成内存峰值的问题。这也解释了为什么之前拿到的vmmap虚拟内存数据里面IOKit是大头。

这样我们找到了关键的线索，在缓存不命中的情况下，大量初始化Image对象，导致IO线程任务繁重，而IO又持有大量的图片解码所用的内存资源。带这个推论，我在Flutter Engine的Task Runner加入了任务数量和C++ image对象的监控代码，证实了的确存在IO任务线程过载的情况，峰值在极端情况下瞬时达到了100+IO操作。

![img](https://www.dengtar.com/res/201903/01/1549936680_4_616.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/640/format/webp)IO Runner监控

到这里问题似乎越来越明了了，但是为什么会有这么IO任务触发呢？上述逻辑虽然可能会有IO线程过载的情况下占用大量内存的情况。上层要求生成新的图片对象，这种请求是没有错误的，设计就是如此。就好比主线程阻塞大量的任务，必然会导致界面卡顿，但者却不是主线程本身的问题。我们需要从源头找到导致新对象创建暴涨真正导致IO线程过载的原因。

#### 大量请求的根源

在前面的线索之下，我们继续寻找问题的根源。我们在实际App操作的过程当中发现，页面Push的越多，图片生成的速度越来越快。也就是说页面越多请求越快，看起来没有什么大问题。但是可见的图片其实总是在一定数量范围之内的，不应该随着页面增多而加快对象创建的频率。我们下意识的开始怀疑是否存在不可见的Image Widget也在不断请求图片的情况。最终导致了Cache无法命中而大量生成新的图片的场景。

我开始调查每个页面的图片加载请求，我们知道Flutter里面万物皆Widget，页面都是是Widget，由Navigator管理。我在Widget的生命周期方法（详细见Flutter官方文档）中加入监控代码，如我所料，在Navigator栈底下不可见的页面也还在不停的Resolve Image，直接导致了image对象暴涨而导致IO线程过载，导致了内存峰值。

看起来，我们终于找到了根本原因。解决方案并不难。在页面不可见的时候没必要发出多余的图片加载请求，峰值也就随之降下来了。再经过一番代码优化和测试以后问题得到了根本上的解决。优化上线以后，我们看到了数据发生了质的好转。
有朋友可能想问，为什么不可见的Widget也会被调用到相关的生命周期方法。这里我推荐阅读Flutter官方文档关于Widget相关的介绍，篇幅有限我这里不展开介绍了。widgets

至此，我们已经解决了一个较为严重的内存问题。内存优化情况复杂，可以点也比较多，接下来我继续简要分享在其它一些方面的优化方案。

## 截图缓存优化

### 文件缓存+预加载策略

我们是采用嵌入式Flutter并使用一套混合栈模式管理Native和Flutter页面相互跳转的逻辑。由于FlutterView在App中是单例形式存在的，我们为了更好的用户体验，在页面切换的过程中使用的截图的方式来进行过渡。

大家都知道，图片是非常占用内存的对象，我们如何在不降低用户体验的同时获得最小的内存消耗呢？假如我们每push一个页面都保存一张截图，那么内存是以线性复杂度增长的，这显然不够好。

内存和空间在大多数情况下是一个互相转换的关系，优化很多时候其实是找一个合理的折中点。
最终我采用了预加载+缓存的策略，在页面最多只在内存中同时存在两个截图，其它的存文件，在需要的时候提前进行预加载。
简要流程图：

![img](https://www.dengtar.com/res/201903/01/1549936680_5_102.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/640/format/webp)简要流程图

这样的话就做到了不影响用户体验的前提下，将空间复杂度从O(n)降低到了O(1)。
这个优化进一步节省了不必要的内存开销。

### 截图额外的优化

- 针对当前设备的内存情况，自适应调整截图的分辨率，争取最小的内存消耗。
- 在极端的内存情况下，把所有截图都从内存中移除存（存文件可恢复），采用PlaceHolder的形式。极端情况下避免被杀，保证可用性的体验降级策略。

## 页面兜底策略

对于电商类App存在一个普遍的问题，用户会不断的push页面到栈里面，我们不能阻止用户这种行为。我们当然可以把老页面干掉，每次回退的时候重新加载，但是这种用户体验跟Web页一样，是用户不可接受的。我们要维持页面的状态以保证用户体验。这必然会导致内存的线性增长，最终肯定难免要被杀。我们优化的目的是提高用户能够push的极限页面数量。

对于Flutter页面优化，除了在优化每一个页面消耗的内存之外，我们做了降级兜底策略去保证App的可用性：在极端情况下将老页面进行销毁，在需要的时候重新创建。这的确降低了用户体验，在极端情况下，降级体验还是比Crash要好一些。

![img](https://www.dengtar.com/res/201903/01/1549936680_6_270.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/640/format/webp)image

## FlutterViewController 单例析构

另外我想讨论的一个话题是关于FlutterViewController的。目前Flutter的设计是按照单例模式去运行的，这对于完全用Flutterc重新开发的App没有太大的问题。但是对于混合型App，多出来的常驻内存确实是一个问题。

实际上，Flutter Engine底层实现是考虑到了析构这个问题，有相关的接口。但是在Embeder这一层（具体FlutterViewController Message Channels这一层），在实现过程中存在一些循环引用，导致在Native层就算没有引用FlutterViewController的时候也无法释放.

![img](https://www.dengtar.com/res/201903/01/1549936680_7_578.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/640/format/webp)FlutterViewController引用图

我在经过一段时间的尝试后，算是把循环引用解除了。这些循环引用主要集中在FlutterChannel这一块。在解除之后我顺利的释放了FlutterViewController，可以明显看到常驻内存得到了释放。但是我发现释放FlutterViewController的时候会导致一部分Skia Image对象泄漏，因为Skia Objects必须在它创建的线程进行释放(详情请参考skia_gpu_object.cc源码)，线程同步的问题。关于这个问题我在GitHub上面有一个issue大家可以参考。FlutterViewController释放issue

目前，这个优化我们已经反馈给Flutter团队，期待他们官方支持。希望大家可以一起探索研究。

## 进一步探讨

除此之外，Flutter内存方面其实还有比较多方面可以去研究。我这里列举几个目前观察到的问题。

1. 我在内存分析的时候发现Flutter底层使用的boring ssl库有可以确定的内存泄漏。虽然这个泄漏比较缓慢，但是对于App长期运行还是有影响的。我在GitHub上面提了个issue跟进，目前已有相关的人员进行跟进。SSL leak issue
2. 关于图片渲染，目前Flutter还是有优化空间的，特别是图片的按需剪裁。大多数情况下是没有不要将整一个bitmap解压到内存中的，我们可以针对显示的区域大小和屏幕的分辨率对图片进行合理的缩放以取得最好的性能消耗。
3. 在分析Flutter内存的MemGraph的时候，我发现Skia引擎当中对于TextLayout消耗了大量的内存.目前我没有找到具体的原因，可能存在优化的空间。

## 结语

在这篇文章里，我简要的聊了一下目前团队在Flutter应用内存方面做出的尝试和探索。短短一篇文章无法包含所有内容，只能推出了几个典型的案例来作分析，希望可以跟大家一起探讨研究。欢迎感兴趣的朋友一起研究，如有更好的想法方案，我非常乐意看到你的分享。





## 参考资料

- （https://github.com/flutter/flutter）Flutter 开发文档
- （https://github.com/flutter/engine）Flutter 引擎文档
- （https://flutter.io/）Flutter IO
- （https://www.dartlang.org/）Dart 语言开发文档

