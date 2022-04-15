# [iOS内存深入研究](https://xilankong.github.io/ios%E6%80%A7%E8%83%BD%E4%BC%98%E5%8C%96/2019/07/05/iOS%E5%86%85%E5%AD%98%E6%B7%B1%E5%85%A5%E7%A0%94%E7%A9%B6.html)



## 引言

对于我们的 App 所依赖的设备而言，内存资源是有限的。降低 App 所使用的内存可以提高性能和体验，相反，过大的内存占用可能会导致 App 被系统强制退出。所以每个 iOS 开发者都应该关注内存问题。这一节新的内容不多，基本上都是一些老的知识点。

按照 Session 的套路，我们先看一下提纲：

- 为什么要减少内存使用
- 内存占用
- 分析内存占用工具
- 图像
- 在后台时，对内存优化
- 演示 Demo

那我们就按顺序开始啦！

## 为什么要减少内存

在探讨内存之前，我们要知道为什么要减少内存。简单的回答是可以有更好的用户体验：更快的启动速度，不会因为内存过大而导致 Crash，可以让 App 存活更久等。

## 内存占用

并非所有 App 的内存占用都是相同的。在继续探讨 iOS 上 App 的内存使用之前，我们先来聊一下`Pages Memory`。

### Pages Memory

内存是由系统管理，一般以页为单位来划分。在 iOS 上，每一页包含 16KB 的空间。一段数据可能会占用多页内存，所占用页总数乘以每页空间得到的就是这段数据使用的总内存。

![img](https://xilankong.github.io/resource/iOS-memory-1.png)

内存页按照各自的分配和使用状态，可以被分为 `Clean` 和 `Dirty` 两类。

![img](https://xilankong.github.io/resource/iOS-memory-2.png)

以上面的代码为例，申请一块长度为 80000 字节的内存空间，按照一页 16KB 来计算，就需要 6 页内存来存储。

- 当这些内存页开辟出来的时候，它们都是 `Clean` 的
- 当向处于第一页的内存写入数据时，第一页内存会变成 `Dirty`
- 当向处于最后一页的内存写入数据时，这一页也会变成 `Dirty`

![img](https://xilankong.github.io/resource/iOS-memory-3.png)

### 内存映射文件

当 App 访问一个文件时，系统内核会负责调度，将磁盘上的文件加载并映射到内存中。如果这是只读的文件，它所占用到的内存页是 `Clean` 的。

如下图所示，一个 50KB 的图片被加载到内存中时，需要分配 4 页内存来存储。其中第四页中有 2KB 的空间会被用来存储这个图片的数据，剩余空间可能会被用来存储其它数据。

![img](https://xilankong.github.io/resource/iOS-memory-4.png)

### 典型app内存类型

当内存不足的时候，系统会按照一定策略来腾出更多空间供使用，比较常见的做法是将一部分低优先级的数据挪到磁盘上，这个操作称为 `Page Out`。之后当再次访问到这块数据的时候，系统会负责将它重新搬回内存空间中，这个操作称为 `Page In`。

然而对于移动设备而言，频繁对磁盘进行IO操作会降低存储设备的寿命。从 iOS7 开始，系统开始采用压缩内存的办法来释放内存空间，被压缩的内存称为 `Compressed Memory`。下面依次介绍一下 iOS App 通常情况下的三种内存类型：`Clean Memory` 、`Dirty Memory`以及`Compressed Memory`。

#### Clean Memory

`Clean Memory` 是指那些可以用以 `Page Out` 的内存，包括已被加载到内存中的文件，或者是 App 所用到的 frameworks。每个 frameworks 都有 `_DATA_CONST` 段，当 App 在运行时使用到了某个 framework，它所对应的 `_DATA_CONST` 的内存就会由 Clean 变为 Dirty。

#### Dirty Memory

`Dirty Memory` 是指那些被 App 写入过数据的内存，包括所有堆区的对象、图像解码缓冲区，同时，类似 `Clean memory`，也包括 App 所用到的 frameworks。每个 framework 都会有 `_DATA` 段和 `_DATA_DIRTY` 段，它们的内存是 `Dirty` 的。

值得注意的是，在使用 framework 的过程中会产生 `Dirty Memory`，使用单例或者全局初始化方法是减少 `Dirty Memory` 不错的方法，因为单例一旦创建就不会销毁，全局初始化方法会在 class 加载时执行。

#### Compressed Memory

当内存吃紧的时候，系统会将不使用的内存进行压缩，直到下一次访问的时候进行解压。

例如，当我们使用 `Dictionary` 去缓存数据的时候，假设现在已经使用了 3 页内存，当不访问的时候可能会被压缩为 1 页，再次使用到时候又会解压成 3 页。

### Memory Warnings

并非所有内存警告都是由 App 造成的，例如在内存较小的设备上，当你接听电话的时候也有可能发生内存警告。按照以往的习惯，你可能会在收到内存警告通知的时候去做一些释放内存的事情。然而内存压缩机制会使事情变得复杂。我们来看看这个例子：

![img](https://xilankong.github.io/resource/iOS-memory-5.png)

假设代码中的 `cache` 已被压缩过

![img](https://xilankong.github.io/resource/iOS-memory-6.png)

事实上，当你尝试去再次访问 cache 对象的时候，系统会先解压这块内存

![img](https://xilankong.github.io/resource/iOS-memory-7.png)

这个过程中内存使用会增加，在内存吃紧的时候，这并不是我们想要的。随后，当我们会执行大量工作去清空 cache，最终得到的内存空间和内存压缩的结果一样

![img](https://xilankong.github.io/resource/iOS-memory-6.png)

所以，相比以往的缓存手段，更加建议去调整策略，例如减少缓存使用，或者在收到内存警告的时候，将这类事情交由系统去处理。

### Caching

我们对数据进行缓存的目的是想减少 CPU 的压力，但是过多的缓存又会占用过大的内存。由于内存压缩机制的存在，我们需要根据缓存数据大小以及重算这些数据的成本，在 CPU 和内存之间进行权衡。

在一些需要缓存数据的场景下，可以考虑使用 `NSCache` 代替 `NSDictionary`，因为 `NSCache` 可以自动清理内存，在内存吃紧的时候会更加合理。

### 小结

通常情况下，我们所说的内存占用是指 `Dirty Memory` 和 `Compressed Memory`，`Clean Memory` 不需要过多关心。

![img](https://xilankong.github.io/resource/iOS-memory-8.png)

App 能使用比较多的内存空间，但是上限会根据设备不同而不同。Extension 能使用的最大内存则要低很多，所以当你在开发 Extension 的时候尤其要注意内存使用。当使用的内存超出限制的时候，系统会抛出 `EXC_RESOURCE_EXCEPTION` 异常。

## 分析内存占用工具

### Xcode Memory Gauge

在 Xcode 中，你可以通过 `Memory Gauge` 工具，很方便快速的查看 App 运行时的内存情况，包括内存最高占用、最低占用，以及在所有进程中的占用比例等。如果想要查看更详细的数据，就需要用到 `Instruments` 了。

![img](https://xilankong.github.io/resource/iOS-memory-9.png)

### Instruments

在 `Instruments` 中，你可以使用 `Allocations`、`Leaks`、`VM Tracker` 和 `Virtual Memory Trace` 对 App 进行多维度分析。

### Debug Debugger-Memory Resource Exceptions

当你使用 Xcode 10 以前的版本进行调试时，在内存过大时，debug session 会直接终止，并且在控制台打印出异常。从 Xcode 10 开始，debugger 会自动捕获 `EXC_RESOURCE RESOURCE_TYPE_MEMORY` 异常，并断点在触发异常抛出的地方，十分方便定位问题。

### Xcode Debug Memory Graph

![img](https://xilankong.github.io/resource/iOS-memory-10.png)

通过这个工具，可以很直观地查看内存中所有对象的内存使用情况，以及相互之间的依赖关系，对定位那些因为循环引用导致的内存泄露问题十分有帮助。

你也可以点击 `File->Export Memory Graph` 将其导出为 `memgraph` 文件，在命令行中使用 `Developer Tool` 对其进行分析。使用这种方式，你可以在任何时候对过去某时的 App 内存使用进行分析。

简单介绍一下相关的命令

#### vmmap - 查看虚拟内存

![img](https://xilankong.github.io/resource/iOS-memory-11.png)

查看详细报告

> vmmap xx.memgraph

查看摘要报告

> vmmap –summary xx.memgraph

配合管道命令查看所有动态库的Ditry Pages的总和

> | vmmap -pages xxx.memgraph | grep ‘.dylib’ | awk ‘{sum += $6} END { print “Total Dirty Pages:”sum}’ |
> | ------------------------- | ------------- | ------------------------------------------------------ |
> |                           |               |                                                        |

只显示CG image相关的数据

> | vmmap xx.memgraph | grep ‘CG image’ |
> | ----------------- | --------------- |
> |                   |                 |

更多使用方式请查看vmmap的文档

> man vmmap

#### leaks - 查看泄漏的内存

查看是否有内存泄露

> leaks xx.memgraph

查看某处内存的泄漏

> leaks –traceTree [内存地址] xx.memgraph

更多使用方式请查看 leaks 的文档

> man leaks



#### heap - 查看堆区内存

查看所有堆区对象的内存使用

> heap xx.memgraph

默认情况下是按照对象数量进行排序，通常情况下它们不会造成什么内存问题。我们需要关心的是那些为数不多，却占用了大量内存的对象，这时候就可以增加参数 `-sortBySize`，按照内存占用大小顺序来查看所有堆区对象的内存使用

> heap xx.memgraph -sortBySize

当确定是哪个类型的对象占用了太多内存之后，可以得到每个对象的内存地址

> | heap xx.memgraph -addresses all | ‘XXBigData’ |
> | ------------------------------- | ----------- |
> |                                 |             |

更多使用方式请查看 heap 的文档

> man heap

有了这些对象的内存地址之后，我们还需要另一样工具帮助我们做下一步分析。

#### Enabling Malloc Stack Logging

在 `Product -> Scheme -> Edit Scheme -> Diagnostics` 中，开启 `Malloc Stack` 功能，建议使用 `Live Allocations Only` 选项

之后 lldb 会记录调试过程中对象创建的堆栈，配合 `malloc_history` 工具，就可以定位到那些占用了过大内存的对象是哪里创建的。

#### malloc_history - 查看内存分配历史

> malloc_history xx.memgraph [address]

> malloc_history xx.memgraph –fullStacks [address]

更多使用方式请查看 malloc_history 的文档

> man malloc_history

### 选择哪个工具？

上面讲述了那么多的分析工具，那我们应该选择哪种工具呢？苹果的工程师帮我们做了如下整理：

![img](https://xilankong.github.io/resource/iOS-memory-12.png)

大家可以根据上图所示，根据不同的需要进行选择。

## 图片

对于 iOS 系统而言，绝大部分场景下哪类数据占内存最多呢？当然是图片！需要注意的是，图片所占内存的大小与图片的尺寸有关，而不是图片的文件大小。

例如：有一个 590KB 的图片，分辨率是 2048px * 1536px，它实际使用的内存不是 590KB，而是`2048 * 1536 * 4 = 12 MB`。。

图片为什么会占用这么大的内存呢，这还要从图片在 iOS 上显示的原理说起，具体可移步到 WWDC 2018 Session 219：[Image and Graphics Best Practices](https://link.juejin.im/?target=https%3A%2F%2Fdeveloper.apple.com%2Fvideos%2Fplay%2Fwwdc2018%2F219%2F)，也可以直接阅读小伙伴前几天刚发布的文章 [WWDC2018 图像最佳实践](https://juejin.im/post/5b1a7c2c5188257d5a30c820)

### 图片的格式

- sRGB：这个是目前比较通用的全色彩图像色域，每个像素占 4 个字节
- Wide：每个像素占 8 个字节，相比 sRGB 能表示的颜色更多

还有占内存更小的格式：

- 亮度和 alpha 8 格式：每像素 2 个字节，单色图像和 alpha，metal 着色器。
- Alpha 8 格式：每个像素 1 个字节，用于单色图像，比 SRGB 小 75％

选择正确的格式可以减少了内存的使用。简单总结一下：

```
一个字节：Alpha 8
两个字节：亮度和alpha 8
四个字节：SRGB
八个字节：Wide 格式
复制代码
```

那下一个话题来了，如何选择正确的格式呢？

### 选择正确的格式

简单的回答是：不需要你来选择格式，而是应该让格式选择你。是不是觉得一下子松了一口气？哈哈😆

#### 用 UIGraphicsImageRenderer 代替 UIGraphicsBeginImageContextWithOptions

使用 `UIGraphicsBeginImageContextWithOptions` 生成的图片，每个像素需要 4 个字节表示。建议使用 `UIGraphicsImageRenderer`，这个方法是从 iOS 10 引入，在 iOS 12 上会自动选择最佳的图像格式，可以减少很多内存。

![img](https://xilankong.github.io/resource/iOS-memory-13.png)

另外，如果想修改颜色，可以直接修改 tintColor，不会有额外的内存开销。

#### Downsampling

当你缩小一幅图像的时候，会按照取平均值的办法把多个像素点变成一个像素点，这个过程称为 `Downsampling`。

UIImage 在设置和调整大小的时候，需要将原始图像加压到内存中，然后对内部坐标空间做一系列转换，整个过程会消耗很多资源。我们可以使用 ImageIO，它可以直接读取图像大小和元数据信息，不会带来额外的内存开销。

![img](https://xilankong.github.io/resource/iOS-memory-14.png)

### 在后台时，对内存优化

假设在 App 里展示了一张很大图片，当我们切换到后台去做其它的操作时，这个图片还在占用内存。我们应该考虑在合适的时机去回收这类占用过大的数据。

![img](https://xilankong.github.io/resource/iOS-memory-15.png)

## 演示Demo

Demo主要是用实际例子讲述了上面的知识点，这里就不再重复讲解了，感兴趣的童鞋可以移步 [iOS Memory Deep Dive](https://link.juejin.im/?target=https%3A%2F%2Fdeveloper.apple.com%2Fvideos%2Fplay%2Fwwdc2018%2F416%2F)

## 总结

内存是一个有限的共享资源，要学会使用 Xcode 分析内存工具，从而了解应用程序内存占用情况，并使用一些缩减应用程序内存占用空间的技巧和窍门。

