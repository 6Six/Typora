# [检测和诊断内存问题](https://mp.weixin.qq.com/s/E80VEIJma66fj7BZy1cCeQ)



## 概览

本 session 讲解了如何使用 Xcode 检测和诊断内存问题。首先需要了解内存构成，内存占用对 app 的影响、以及一些常见的内存问题，最后学习使用一些工具来分析并解决内存问题。

阅读指导：为了保证文章的完整性，我们会对一些概念进行详细的解释，你可以速读已了解的部分，或者直接

跳至下一小节继续阅读。

## 内存占用的组成

在了解内存问题之前，首先让我们先来复习一些内存的基础知识。让我们看看是什么组成了内存占用 (memory profile)。我们用三个类别来对 app 的内存占用进行分类。脏内存 (Dirty memory)，压缩内存 (Compressed memory), 干净的内存 (Clean memory)。让我们快速的看一下每一项都包含什么。

![图片](https://mmbiz.qpic.cn/mmbiz_png/deSLfic6WeGWMF8yPLzD5Rv3s26mceB6qbFetWV1eibicJib5dq96iaRYCrysia0NoLr1w8ftoa0vUKibRCwmJmUUAibFw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)image-20210615001122429

### Dirty memory

Dirty memory 是已经被 app 写入的内存，包含如下：

1. 它包括所有的 heap allocations：当你用 malloc 时，申请的就是堆上的存储空间。
2. 图像解码的 buffer。
3. 以及 frameworks 中的 `__DATA` 和 `__DATA_DIRTY` 部分也同样存储在 Dirty memory。

> Tis: Frameworks you link actually use clean memory and dirty memory

![图片](https://mmbiz.qpic.cn/mmbiz_png/deSLfic6WeGWMF8yPLzD5Rv3s26mceB6qyGI2Xn7ciaiaaOBXu82icwKL8vUetZqFQ2yEqoBibhW3cXibPIMBxKP4NNg/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)image-20210615001213304

### Compressed memory

苹果最初只是公开了从 OS X Mavericks 开始使用 Compressed memory 技术，但 iOS 系统也从 iOS 7 开始悄悄地使用。从 **OSX_Mavericks_Core_Technology_Overview**[2]文档中可以了解到该技术在内存紧张时能够将最近未使用过的内存占用压缩至原有大小的一半以下，并且能够在需要时解压复用。它在节省内存的同时提高了系统的响应速度，其特点可以归结为：

- Shrinks memory usage 减少了不活跃内存占用
- Improves power efficiency 改善电源效率，通过压缩减少磁盘IO带来的损耗
- Minimizes CPU usage 压缩/解压十分迅速，能够尽可能减少 CPU 的时间开销
- Is multicore aware 支持多核操作

Compressed memory 是将 Dirty memory 中最近没有访问过得内存，使用内存压缩器对 Dirty page 进行压缩。这些 page 会在被访问时解压缩。注意 iOS 是没有交换内存(Disk swap)技术的，交换内存是 MacOS 特有的。

> Disk swap 是指在 macOS 以及一些其他桌面操作系统中，当内存可用资源紧张时，系统将内存中的内容写入磁盘中的backing store (Swapping out)，并且在需要访问时从磁盘中再读入 RAM (Swapping in)。与大多数 UNIX 系统不同的是，macOS 没有预先分配磁盘中的一部分作为 backing store，而是利用引导分区所有可用的磁盘空间。

> iOS 在内存紧张的时候会使用到内存压缩技术，而MacOS在内存紧张的时候会使用到内存压缩技术及磁盘交换技术

![图片](https://mmbiz.qpic.cn/mmbiz_png/deSLfic6WeGWMF8yPLzD5Rv3s26mceB6qU5EpZd1GdCQyqdMNMCQZ6LTmTZK0vUB1NZV8j5TukLNyuSFr01LicMw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)image-20210615001236922

### Clean Memory

Clean Memory 是还没有被写入的内存或可以被 page out 的内存。指的是还没有被加载到内存或者能够被系统清理出内存且在需要时能重新加载的数据。包括：

- Memory mapped files (内存映射文件)
- 加载到内存中的磁盘上的图像
- Frameworks 中的 __DATA_CONST 部分
- 应用的二进制可执行文件

![图片](https://mmbiz.qpic.cn/mmbiz_png/deSLfic6WeGWMF8yPLzD5Rv3s26mceB6qn9iaAHxSX5tz6ETmzdUp1e2HVDC3Jm2MraEMfy03Tr8DCBXLkFA8ohA/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)image-20210615001328356

**因此， memory footprint = dirty size + compressed size ，也就是我们需要尝试去减少的内存占用。**

如果想对内存技术有更深刻的了解，建议观看 **WWDC18 iOS Memory Deep Dive**[3]。

## 内存占用的影响

那么，为什么我们需要关注应用的内存占用 (Memory footprint)？

**为了更好的用户体验。**

[^Memory footprint]: 这里的 Memory Footprint 指的是：Dirty + Compressed。

> Your app's memory footprint consists of the data that you allocated in RAM, and that must stay in RAM (or the equivalent) at all times.

> **RAM**[4]:随机存取存储器（英语：Random Access Memory）是与 **CPU**[5] 直接交换数据的内部存储器

**合理的利用内存，可以从四个方面来提升用户体验**

![图片](https://mmbiz.qpic.cn/mmbiz_png/deSLfic6WeGWMF8yPLzD5Rv3s26mceB6qX7fYN9Y3ffXLOpLD4TZejWnR0tXkZxsoW0icxh3aakox0HLyuDFCRqg/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)image-20210613150748853

### Faster application activations(更快的应用程序激活)

系统的内存是有限的，如果你的 app 切换到后台却占用大量的内存空间时，很有可能被系统终止你的 app 的运行来回收内存空间。所以**我们应该在 App 进入后台时释放内存占用较大的资源，进入前台时重新加载**。因为资源加载到内存是需要时间的，所以保持内存占用的紧凑，可以有效提高你的 app 保留在内存的几率，这样你就能够获得更快的应用程序激活。

> Tips：如何在 app 进入后台时释放内存占用较大的资源，进入前台时重新加载。请参考 **WWDC18 iOS Memory Deep Dive**[6] Optimizing when in backgroud 部分。

### Responsive experience(快速响应的经验)

当用户浏览你的新功能时，他们想要更快速的响应。而减少内存占用就可以让你的 app 获得更快的响应。慎重考虑一下 app 加载到内存的内容，可以有效地减少用户在和你的 app 交互时,系统对内存的回收。

### Complex  features(复杂的功能)

对内存使用采取有效策略，这样节省下来的内存可以让你为 app 增加更多复杂的功能，比如加载视频，做动画等等。

### Wider device compatibility(广泛的设备的兼容性)

最后，Apple 的设备会随着时间不断发展，新设备拥有比以前更多的物理内存。通过减少内存占用，你的应用在旧设备上的表现依旧会很好，从而增加欣赏你的应用的用户。

**总结一下：**

通过监控你 app 的内存占用，你的 app 将获得**更快的应用程序激活**，**更快速的响应**，**处理更复杂的功能**，**能在更多的设备上运行**。

## 常见的内存问题

既然内存对 App 的体验如此重要，那么常见的内存问题有哪些呢？

- Leaks
- Heap size issues

### Leaks

内存泄露是常见的内存问题，它还可以被细分成：

- Allocated objects to which there are no active references 对象失去了引用，却还存活着
- Retain cycles 循环引用

为了方便读者理解，我们会对这两种情况进行更加详细的解释，如果你确认你已经十分了解这两种情况，请跳到 3.2 小节继续阅读。

#### Allocated objects to which there are no active references

![图片](https://mmbiz.qpic.cn/mmbiz_png/deSLfic6WeGWMF8yPLzD5Rv3s26mceB6qibKSHAiaDCX3kFrOiaLY3jxRWjToRz4Neaiaibt7y54lfW8bEM5HKSAjHOQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)image-20210615105635144

当进程创建了对象,在失去了所有指向该对象的指针的时候，并没有回收该对象。我们称这种情况为泄露 (Leak)。我们用灰色的箭头表示对象之间的引用，每个对象都有至少有一个引用。

![图片](https://mmbiz.qpic.cn/mmbiz_png/deSLfic6WeGWMF8yPLzD5Rv3s26mceB6qbwD8ibibeZxeNk7aFUestI8Vt4TyV5Er6woYBFYWdeWgyAibUSSwFKU8A/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)image-20210615105830292

注意 A 和 B 之间的虚线，这表示此时我们把 A 对 B 的引用置为 nil，并且移除 A 上的 B 对象。

![图片](https://mmbiz.qpic.cn/mmbiz_png/deSLfic6WeGWMF8yPLzD5Rv3s26mceB6qydR0ia0148IJA20ORgo30LBc34jzfFDaIFXQfXH56NDNNriaj5dwTOug/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)image-20210615110643754

当指针被移除时，B 对象就泄露了。已经没有任何对 B 的引用，但是 B 对象依旧被存储在 Dirty Memory 中。但是进程中已经没有引用指向它了，也就没有办法去释放它。当泄露的对象越多时，他们所占用的 Dirty Memory 就越多，所以我们需要修复泄露。

> 这种情况的泄露，只会发生在 MRC 下，在 ARC 下当移除指针时，我们一般就会认为此对象已经被释放。

#### Retain cycles

![图片](https://mmbiz.qpic.cn/mmbiz_png/deSLfic6WeGWMF8yPLzD5Rv3s26mceB6qVkYtTteNUWzoJ5uYcpYCiaxUmaFbkRG0Rr38Ar70soAHlzGGQ9lA6ibw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)image-20210615112430981

循环引用也会引起泄露。Swift 中的最常见对象泄露就是循环引用引起的。在上图中，对象 A 和 B 就是循环引用。它们相互引用，但没有外部引用。这意味着进程不能访问或者释放他们。所以它们被认定为泄露。

![图片](https://mmbiz.qpic.cn/mmbiz_png/deSLfic6WeGWMF8yPLzD5Rv3s26mceB6qL7wkL92fbBqiblhdegsm52K5K2E7MujEB4kEGhhHuVsr90tiaqjviaNdA/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)image-20210615151156529

幸运的是，大多数的 swift 对象都被  Swift 的自动引用计数系统 ( Swift's automatic reference counting system)或者 ARC 管理，这样可以阻止大部分的泄露。如果你用 ARC 管理对象，需要注意 unsafe 类型，确保你会在失去所有引用之前去释放它们。即使是 ARC 管理的对象，也容易变成循环引用。所以，避免创建循环强引用，如果这个循环引用是绝对必要的，考虑使用  `weak` 引用代替强引用，因为  `weak` 引用不会阻止对象被回收。

### Heap size issues

堆是进程地址空间的一部分，用来存储动态生成的对象。所以 堆的大小也对内存占用起到了至关重要的影响。为了保证程序的运行，我们无法避免的要在堆上生成对象，那么这些对象该如何有效的治理呢？

那么首先我们需要确定堆上容易出现哪些问题？

- Heap allocation regressions 堆分配回归
- Fragmentation 碎片化

下面我们会分析这些问题的成因，以及对应的治理策略。

#### Heap allocation regressions

![图片](https://mmbiz.qpic.cn/mmbiz_png/deSLfic6WeGWMF8yPLzD5Rv3s26mceB6qwv24Y8UKUjvgiaIuDIPNt1l1XnsyRSY6ENbeGicCsF1NyOWddbic0PwZw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)image-20210615204512092

堆只是进程地址空间的一部分，用来存储动态生成的对象。堆分配回归会增加内存占用，因为进程在堆上比以前生成了更多对象。为了减少堆的回归，可以删除无用分配并缩小不必要的大内存分配。你也应该关注一下你一次持有多少内存。释放掉你不在使用的内存，并在你需要的时候才去分配内存。这将减少 app 的内存峰值。让它被终止的几率变得更小。

总结一下 Heap allocation regressions  对应的治理策略：

- 移除无用内存分配。
- 减少过大内存的分配。
- 不再使用的内存需要释放。
- 在你需要的时候，才去分配内存。

#### Fragmentation

碎片带来了碎片化的问题，那么碎片是如何产生的？首先让我们快速回顾一下 page 在 iOS 中是怎样工作的。

![图片](https://mmbiz.qpic.cn/mmbiz_png/deSLfic6WeGWMF8yPLzD5Rv3s26mceB6qBxdIcp8RaKmeTMAeFlgvJ08sDwEkZHWMG8UI74q5ruZIoJstm5Qk1g/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)image-20210616215204050

page 是系统授予进程的固定大小、不可分割的最小内存块。因为 page 是不可分割的，当进程写入 page 的任意部分，整个 page 都会被认为是 dirty 的并且进程将会管理它，即使 page 的大部分没有被使用到。

当进程的 dirty page 没有被 100% 占用时，就会产生碎片化。为了理解为什么出现碎片，我们来看一个例子：![图片](https://mmbiz.qpic.cn/mmbiz_png/deSLfic6WeGWMF8yPLzD5Rv3s26mceB6qHxichbTh6T1NEldWDMvQeI6glWlGyASzsF4zT7aSicRuyE4MFbtYLibKQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

首先有三页 clean page。

![图片](https://mmbiz.qpic.cn/mmbiz_png/deSLfic6WeGWMF8yPLzD5Rv3s26mceB6q2OTgic4LD6w4pp9k2dWnpQJgqbAsU4rKXKpickWwZNWrX2gmoRlzESfQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)image-20210616215948349

当进程运行的时候，创建的对象会填满这些 page。此时 clean page 就变成了 dirty page。

![图片](https://mmbiz.qpic.cn/mmbiz_png/deSLfic6WeGWMF8yPLzD5Rv3s26mceB6qZdQmF5XiciaSJia8icQV4PaMgGqLxjOaCD8V1ZicSCrLjeThr4RWfQAfMjQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)image-20210616220217236

当部分对象被释放,它们填充过得地方就会变成空槽,在上图中被标记为 free memory。因为依旧填充着对象，这两个 page 依旧被标记为 dirty。

![图片](https://mmbiz.qpic.cn/mmbiz_png/deSLfic6WeGWMF8yPLzD5Rv3s26mceB6q17BxYcCr248lNDRmdooZUKPq4ztyIIS9jEHtib6OTIQibPniaj4vNUW7Q/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)image-20210616220624885

系统想用将要创建的对象填充空槽，右侧蓝色方块是即将要创建的对象，不幸的是，即将创建的对象太大而不能插到空槽里，即使空槽的大小加起来足够大，但是空槽不是连续的。它们不能给一整个对象使用。

![图片](https://mmbiz.qpic.cn/mmbiz_png/deSLfic6WeGWMF8yPLzD5Rv3s26mceB6q1HVgwrJC2QTYtibib9es9z1nF2hUxzSCv6USbIChCSogZFiazQ0ZjiazUg/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)image-20210616222503351

因为现有的空槽不能使用，系统就会启用一个新的 page 给即将要创建的对象。如上图最右侧的方格就是新的  page，现有的内存空槽依旧没有没填满， 这种情况我们就称之为**碎片化内存**。

**最好解决内存碎片化的方法就是创建内存相邻，生命周期相似的对象**。这能帮助确保所有这些对象会被一起释放，这样进程就会得到一大块连续的空闲内存来为即将要被创建的对象服务。

总结一下解决内存碎片化的方法：

- 创建内存相邻，生命周期相似的对象，这样在这些对象释放之后，我们就会得到一大块连续的空闲内存

## 内存治理的工具

既然内存中会有这么多的问题，我们又不可能在开发代码的阶段就**完全避免**这些问题，苹果为了让我们可以有效的检测和诊断这些内存问题，开发了一系列的工具来帮助开发者，下面让我们来谈谈这些工具。

### 已有的内存治理工具

内存问题由来已久，苹果在今年之前就有很多工具可以帮我们来检测和诊断内存问题，我们简单的把已有工具在使用维度上分为：

- 可视化工具
- 命令行工具

下面我们会详细的列举这些工具，并且简单的阐述一下这些工具的优缺点，以及组合使用方案，因为一些工具存在的时间比较长，笔者并不能一定能找到对应工具组合的最优解，如果你知道，请在评论区留言，如果你的方案更好，我们会更新到文章中。

#### 可视化工具

可视化工具又分为：

- Xcode 集成的工具
- instruments 相关工具

#### Xcode 集成的工具：

- Memory Report

  ![图片](https://mmbiz.qpic.cn/mmbiz_png/deSLfic6WeGWMF8yPLzD5Rv3s26mceB6qkf95UN9zSJoP44XUz8vWBjIIy78hribbDBqrDGtEENy4RS52VsBQl4w/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)Memory Report

  Memory Report 存在 Debug navigator 中，当**程序运行起来**，切换到  Debug navigator 点击 memory 就可以查看 Memory Report , 这个报告只能粗略的查看内存状况，比如：**通过 push 出一个 controller 查看对应的内存增长，pop 掉这个 controller 之后一般会有对应的内存减少**。当然如果这个 controller 存在大量的网络图片展示，就比较特殊了，一般的网络图片下载和缓存框架为了减少磁盘 IO 以及提高多次访问图片的命中率，会对**进行图片缓存**，这时 push 的内存增长和 pop 内存减少就是不对称的状态。比如 `SDWebImage` 会在程序切换到后台的时候，会释放掉一部分缓存。你可以通过切换到后来来验证，当前的不对称是都由网络图片的缓存造成的。所以说 Memory Report 是一个更加整体的内存概况，比较适合查看内存概况，以及没有网络图片缓存的 controller 的释放情况。

  **优势**：快速查看整体内存预览。

  **短板**：内存概况不够详细，即使查看对应 controller 的创建以及释放都有一定的局限性。

- Product->Analyze

Product 中的静态分析主要分析以下四种问题：

a.) 逻辑错误：访问空指针或未初始化的变量等

b.) 内存管理错误：如内存泄漏等

c.) 声明错误：从未使用过的变量

d.) Api调用错误：未包含使用的库和框架

注意使用静态分析是基于编译器的静态检查，而 `Objective-C` 是具有相当强大的动态性，所以静态分析能够检查出一些内存泄露问题，一些动态执行引起的内存泄露需要其他工具来检查。

**优势**：静态分析是基于编译器的静态检查，且检查会涵盖多种问题的检查。

**短板**：静态检查本事是基于静态的检查，对应 `Objective-C` 这种动态性语言的检查具有一定的局限性。

- Schemes 的诊断工具中的 Memory Management

- - Malloc Scribble

    > 申请内存后在申请的内存上填 `0xAA`，内存释放后在释放的内存上填 `0x55`；再就是说如果内存未被初始化就被访问，或者释放后被访问，就会引发异常，这样就可以使问题尽快暴漏出来。
    >
    > `Scribble` 其实是 `malloc` 库 `libsystem_malloc.dylib` 自身提供的调试方案

  - Malloc Guard Edges

    > 申请大片内存的之前或者之后都会在 page 上加保护

  - Guard Malloc

    > 使用 `libgmalloc` 捕获常见的内存问题，比如越界、释放之后继续使用。
    >
    > 由于 `libgmalloc` 在真机上不存在，因此这个功能只能在模拟器上使用.

    Guard edge 和 Guard Malloc 可以帮助你发现内存溢出，并在通过对申请的大块内存保护和延迟释放来使你的程序在误用内存时产生更明确地崩溃。

  - Zombie Objects

    > `Zombie` 的原理是用生成僵尸对象来替换 `dealloc` 的实现，当对象引用计数为 `0` 的时候，将需要`dealloc` 的对象转化为僵尸对象。如果之后再给这个僵尸对象发消息，则抛出异常，并打印出相应的信息，调试者可以很轻松的找到异常发生位置。

  - Malloc Stack logging

    Malloc Stack logging 可以结合 Debug Memory Graph 进行使用，我们会在 Debug Memory Graph 处更加详细的说明 Malloc Stack logging 的作用。

    **诊断工具 Memory Management 总结**：

    **优势**：诊断工具 Memory Management  更加聚焦于最基础的内存使用，包括涂鸦，page 边界保护，越界以及对已经释放的地址进行访问等。

    **短板**：部分会存在模拟器的限制，因为这块比较聚焦基础的内存，部分功能对开发者的要求也比较高。

    **注意**：Memory Management 的这五个工具是在对应的 scheme 上生效的，如果你不想 dirty 公共的工程配置，一般可以 选择 `Duplicate Scheme` 并且取消 `share`选项的勾选。而且 Malloc Stack logging 会在你使用 Debug Memory Graph 之后，记录很多日志，增大 app 的沙盒占用，会耗掉手机很多的磁盘空间。建议使用完成之后及时关闭 Malloc Stack logging 。

- Debug Memory Graph

  ![图片](https://mmbiz.qpic.cn/mmbiz_png/deSLfic6WeGWMF8yPLzD5Rv3s26mceB6qcCFN0yrVnqtgX9yNw3ibyHXX1dfmib5lOxBRia5lE5m3PDIiaKmlbofvQQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)image-20210614231737001

  **基础使用**：

  Xcode 运行起 app 之后，在调试栏点击 Debug Memory Graph ，这是 Xcode 会捕获当前 app 的内存快照，此时你可以很方便的查看内存中的存活对象，以及从 app 启动到此刻产生的内存泄露（紫色的叹号代表内存泄露），你可以灵活的选择展示当前内存内所有的存活对象，内存泄露的对象，也可以屏蔽系统的存活对象只关注当前工程调用产生的对象，或者是基于上述的选择，筛选指定类型对象。筛选之后，你可以看到当前类型对象有多少个，点击某个对象可以查看它的引用关系，右侧的 inspectors 还会展示当前对象的详细信息，比如占用大小，调用堆栈等。

  **进阶使用**：

  ![图片](https://mmbiz.qpic.cn/mmbiz_png/deSLfic6WeGWMF8yPLzD5Rv3s26mceB6qJ9ic88EdpufuaJln3jZk8Lm53P3vqP2YDfZ7BdkzyUhWn9IMkTU0maw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)malloc stack logging

  如果你开启 Malloc Stack logging，选择 All Allocation and Free History 选项，你则可以通过调用堆栈直接锚定到具体的代码了。

  如果你需要记录当前内存以备后续分析，你可以在 Xcode 的 File 选项下，导出 **memgraph** 。Xcode 使用 memgraph 的文件格式来储存应用程序的占用信息，导出 memgraph 文件可以结合命令行工具进行分析。

#### instruments 相关工具：

- leaks

用于检测程序运行过程中的内存泄露，并记录对象的历史信息。

- Allocations

  追踪程序的虚拟内存占用和堆信息，提供对象的类名、大小以及调用栈等信息。

- Zombies

  用于检测程序运行过程中的僵尸对象，并记录对象的产生过程，调用堆栈及位置。

- VM Tracker

  能够区分程序运行时前文所述的各种内存类型占用情况，Instruments User Guide 中给出了各个参数的具体定义。

  **Tips**: 在使用上述工具时，如果看不到类和方法名称，绝大部分原因是你的打包模式没有开启dSYM或者debug symbols。

  因为 instruments 相关工具的使用解释起来需要很长的篇幅，这里我推荐几篇文章方便大家了解这几个工具的使用：**leaks**[7]   **Allocations**[8]   **Zombies**[9]   **VM Tracker**[10] 想要了解更加详细的信息，请参阅 **WWDC19 Getting Started with Instruments**[11] 。

#### 命令行工具

在上面我们已经了解了 Xcode 内置的可视化工具，**虽然可视化工具已经能够直观的表现我们想要了解的内存占用信息，但是在终端中不仅可以灵活地利用各种命令和 flag 突出我们想要的内容，更可以快速的实现信息查找和文本化交互。\**在了解内存问题分类之前我们先简单的了解下\**四种常用的命令行工具**。

- vmmp

  > vmmap 能够打印出进程信息，所有分配给该进程的 VMRegions 以及 VMRegion 的种类、内存占用信息等内容。利用 --summary 则能够根据不同的 region type 打印出详细的内存占用类型和信息。这里需要注意的是 SWAPPED SIZE 在 iOS 上指的是 Compressed memory size 且其值表示压缩前的占用大小。

- leaks

  > leaks 追踪堆中的对象，打印出进程中内存泄露情况、调用堆栈以及循环引用信息。利用 --traceTree 和指定对象的地址，leaks 还能以树形结构打印出对象的相关引用。

- heap

  > heap 会打印出所有在堆上的对象信息，默认按类数量排序，也可以通过 -sortBySize 按大小排序，对于追踪堆中较大的对象十分有帮助。找到目标对象后，通过 -address 获得所有/指定类的地址，继而可以利用 malloc_history 寻找其调用堆栈信息。

- malloc_history

  ```
  malloc_history App.memgraph --fuStacks [address]
  ```

  > 使用上述命令能够获得我们知道地址的对象的调用堆栈信息，它能够得到的比 memory inspector 中 Backtrace 更加详细。但是需要开启 Dignostics 中的 Malloc Stack 选项，才能通过 malloc_history 获得 memgraph 记录的调用堆栈信息。

更多拓展信息请参考 **深入解析iOS内存 iOS Memory Deep Dive**[12] 。

### 新增的内存治理工具

今年苹果为开发者提供了**使用 XCTest 框架进行测试，然后通过生成的 Ktrace file 和 Memory graphs 文件来检测和诊断内存问题**，并且拓展已有的命令行工具的参数来帮助开发者更快的定位到问题。学习使用新工具之前，简单了解一下现在你可以使用到的分析内存占用的一些工具。Xcode 提供了一套工具来协助我们监控开发阶段和线上的 app 内存性能。

![图片](https://mmbiz.qpic.cn/mmbiz_png/deSLfic6WeGWMF8yPLzD5Rv3s26mceB6qtyuXhC9icCB4ndORLrzMdVRZoKl0ggIakZzmAQNFBlpPI488MfYQcVw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)image-20210614163556168

XCTest 框架帮助我们直接在项目的单元测试和 UI 测试中监控 app 的内存占用， MetricKit 和 Xcode Organize 帮助我们自定义的监控生产环境上的内存指标。

![图片](https://mmbiz.qpic.cn/mmbiz_png/deSLfic6WeGWMF8yPLzD5Rv3s26mceB6qWvEduELqG0chGicl7AiaEQicqqWMBLcDlZGpZRAeia3z2OQ51zlxsn1DJA/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)image-20210614165019022

我们将使用 XCTests 做性能测试，但是注意这些技术还可以应用在一般内存问题分类和调查中。使用 XCTests 做性能测试, 你能测量系统资源，比如：内存利用率, CPU 使用率，磁盘写入等等。

![图片](https://mmbiz.qpic.cn/mmbiz_png/deSLfic6WeGWMF8yPLzD5Rv3s26mceB6qAsp1IZpojooHnoVC1JTx2RdGam6IM0YTeW3SuSZfl6mhKbMQ32sKjQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)image-20210614223611948

苹果在Xcode 13 **新增**了使用 **XCTest** 收集诊断数据的新功能，来帮忙分类测试回归。通过执行 **XCTest** 用例来生成 `Ktrace files` 和 `Memory graphs`。

#### Ktrace file

![图片](https://mmbiz.qpic.cn/mmbiz_png/deSLfic6WeGWMF8yPLzD5Rv3s26mceB6qJxJuSEeMficq5GSh9G45hGvPHzsqWBNXW1C2XIKMwl7icd6yrZAbyMLw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)image-20210614231308710

`Ktrace files` 即强大又灵活。它可以用于一般的问题诊断也能聚焦于一些特殊的问题，比如可以深入渲染管线调查 `hitches` 问题，或者查找阻塞主线程并导致挂起的原因。在日常工作中，这些 `Ktrace files` 可以用  instruments 打开并分析。

#### Memory graphs

![图片](https://mmbiz.qpic.cn/mmbiz_png/deSLfic6WeGWMF8yPLzD5Rv3s26mceB6qcCFN0yrVnqtgX9yNw3ibyHXX1dfmib5lOxBRia5lE5m3PDIiaKmlbofvQQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)`Memory graphs` 对于特定的问题查询很有用，`Memory graphs` 即可以在 Xcode 的可视化调试工具中使用，也可以作为多个命令行工具使用。其中一些我们将会在后面讨论。`Memory graphs` 本质上是一份进程地址空间的快照，`Memory graphs` 记录了每一个虚拟内存 region 的地址和大小和每一个分配地址的 block ，以及这些 region 和 blocks 的指向。这些足以支撑你去检查每一个堆上对象，查看与链接框架(Link Framworks)关联的数据区域等等。

XCTest 默认打开 malloc stack 的日志，并捕获新创建对象的堆栈。

![图片](https://mmbiz.qpic.cn/mmbiz_png/deSLfic6WeGWMF8yPLzD5Rv3s26mceB6qvWxUOqMdCGolRttopHqpKHNVsV7w34cwDibH9ic5oicANL8Wt2c8kicfDg/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)image-20210614232529918

为了收集诊断可以使用命令行工具把 enablePerformanceTestsDiagnostics 设置为 YES。这个参数可以让 `Ktrace` 收集非内存指标和内存指标的内存图。

## 如何使用新工具

![图片](https://mmbiz.qpic.cn/mmbiz_png/deSLfic6WeGWMF8yPLzD5Rv3s26mceB6qoC2uiaG6mF2a9Y9CVgRFAhx8tDMe2xsC08GtUjsNw5ibdy5UQnABtXVg/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)image-20210614170804690

苹果提供了一个名为 Meal Planner 的 app 来测量内存的使用情况。当点击保存按钮时，会下载对应的食谱到用户的设备上。case 如下：

```
// Monitor memory performance with XCTests

func testSaveMeal() {
    let app = XCUIApplication()
        
    let options = XCTMeasureOptions()
    options.invocationOptions = [.manuallyStart]
        
    measure(metrics: [XCTMemoryMetric(application: app)],
            options: options) {

        app.launch()

        startMeasuring()

        app.cells.firstMatch.buttons["Save meal"].firstMatch.tap()
            
             let savedButton = app.cells.firstMatch.buttons["Saved"].firstMatch
        XCTAssertTrue(savedButton.waitForExistence(timeout: 30))
    }
}
```

`measure(metrics:options:block:)` 需要指定对应的 app，在 `block` 中 启动 app， 调用 `startMeasuring` 开始测量，点击 `Save meal` 按钮， 使用 `waitForExistence` 等待下载食谱完成，并检查 UI 是否更新。执行测试代码之后，点击测试 case 旁边的菱形，弹出测量面板, 选择物理内存选项。

![图片](https://mmbiz.qpic.cn/mmbiz_png/deSLfic6WeGWMF8yPLzD5Rv3s26mceB6qQxbibibSMJQiaWczLZIT9C1icOl0eexrib3PbE7iaLNyYffwFpUvx5SsZNYw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)image-20210614220821604

我们可以看到 case 执行了五次，内存均值在 116000 KB, Set Baseline 文字下方可以看到 case 每次运行的的详细数据，一般情况下我们会参考平均值设置我们的 baseline，设置完 baseline 后再次运行，可以在 Result 看到回归。

> case 每次执行的结果与 baseline 的偏差称为回归(**regression**[13])

![图片](https://mmbiz.qpic.cn/mmbiz_png/deSLfic6WeGWMF8yPLzD5Rv3s26mceB6q7tj8r4mqdd1fRsjefQ8l8bzkJ3V7XbefqjYUcYMhUs1qyc4IpGr8oQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)image-20210614222606040

如果回归大于 baseline， case 就会执行失败。这时候我们就需要停下来查找问题，并且修复，直到 case 执行成功。

当执行完成之前写的性能测试，我们将看到上面的控制台上的打印。打印的内容非常多，但是可以直接查找几个关键词， 第一个要找到的是我们的执行结果是否通过，这个 case 的执行就没有通过。输出也会指出测试失败是因为回归 (**regression**[14])，新的平均值要比 baseline 糟糕 12%，最后我们能够找到 xcresult bundle 的路径。当我们在 Xcode 打开 xcresult bundle， 我们将看到内存测试在顶部，与测试名称相邻。

![图片](https://mmbiz.qpic.cn/mmbiz_png/deSLfic6WeGWMF8yPLzD5Rv3s26mceB6qIep3WBsSKeV6hGwJyVHzepkXQiaibPqBAaxdbicAsxarZ18W6DU4B51Sw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)展开测试日志，可以找到可以获取的内存图。

![图片](https://mmbiz.qpic.cn/mmbiz_png/deSLfic6WeGWMF8yPLzD5Rv3s26mceB6qzcPRiaILaCdYUEzcW9IzQh3yel6AO7NQDSUp9mDQocJ3ffTOCabTVMw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)image-20210614235423403

下载并且解压后，我们会发现如下两个内存图，苹果收集了最初的内存图并在名称前面添加了 pre，收集了最后一次迭代的内存图，并在名称前添加了 post。我们可以通过前后两个内存快照分析期间的内存增长。

![图片](https://mmbiz.qpic.cn/mmbiz_png/deSLfic6WeGWMF8yPLzD5Rv3s26mceB6qpZQvsOqiaz5JAt1ewNxrShM7ic6c5RBOkFAPaciczG9iaIIRkkvZFdDLBg/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)image-20210615000432894

有了`Ktrace files` 和 `Memory graphs` 并且设置了 malloc stack 为 YES，你不仅可以知道回归(**regression**[15])出现了问题，还可以知道为什么回归 (**regression**[16])会出现问题。

### 检测和诊断内存泄露

![图片](https://mmbiz.qpic.cn/mmbiz_png/deSLfic6WeGWMF8yPLzD5Rv3s26mceB6qwRR9H7YGZMnEw1FEib7Z7KJX2Wpsx7ybcKT4rt2DPMtyDXEB7PUPSNg/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)image-20210615151419176

大家还记得我们之前使用 XCTest 执行 case 失败时生成的两份文件吧，我们将使用这两份文件来检查泄露。

![图片](https://mmbiz.qpic.cn/mmbiz_png/deSLfic6WeGWMF8yPLzD5Rv3s26mceB6qYanvMwAntD0Pvvic6BVxTf6CokHLaiaVfvUW3hiaQUnjwRcc2Puhmroyg/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)image-20210615151716402

```
leaks App.memgraph
```

leaks 命令可以帮查找已经产生的泄露。

![图片](https://mmbiz.qpic.cn/mmbiz_png/deSLfic6WeGWMF8yPLzD5Rv3s26mceB6qYpM4HrtiaJYByjImibJcQPhpmbhF8M3g2AibI6UcdnVGHPDdd3C8tBDtQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)4leaksfor240bytes

输出展示了我们有 4 个泄露，一共 240 byte。

![图片](https://mmbiz.qpic.cn/mmbiz_png/deSLfic6WeGWMF8yPLzD5Rv3s26mceB6qskDcxp1grBMLlzRjR4mfd5XIAibv2YaRxaKVcIbV9mORwIdEic4MgWJA/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)image-20210615152124699

接下来，输出包含了每一个泄露的详细的情况，这些信息可以给我们一些线索，帮助我们找到是什么引起了泄露。最上面的对象图指出 ROOT CYCLE,这表示我们面对的是循环引用。

![图片](https://mmbiz.qpic.cn/mmbiz_png/deSLfic6WeGWMF8yPLzD5Rv3s26mceB6qTKZcwZ8sQt7KfmHYUrbd5icPVgNoZLjyC0ATWtP9V2DO4EzQLC0CqaQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)image-20210615152802392

这里有一些有用的符号，让我们看一下，这个循环引用可能包含 MealPlan 和 MeunItem 对象。因为  malloc stack logging 是对 XCTest 打开的，输出就会包含每一个泄露的创建堆栈。这个对于定位是哪个对象产生了泄露真的很有用。

![图片](https://mmbiz.qpic.cn/mmbiz_png/deSLfic6WeGWMF8yPLzD5Rv3s26mceB6qAz8tjPPIPgbGF6m5OYxfGYK8E8U72JlP7aT1sgJem2VTUB7eeLHAEg/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)image-20210615181249641

通常，你会希望从代码中找到具有符号的调用堆栈,这个是我代码中的一部分调用。正在泄露的 MealPlan 对象是在 populateMealData 中创建的。我们打开 Xcode 来看一下我们是否能够修复这个问题。

![图片](https://mmbiz.qpic.cn/mmbiz_png/deSLfic6WeGWMF8yPLzD5Rv3s26mceB6qpSTDSdWrLYUbpnnsLGmDbicBeQEo7UMCkaKNqbSYbl16EphGzn3iaeKg/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

这个函数就是我们在 leaks 的输出中看到的，这里我分别创建了 MealPlan 对象和 MealItem 对象，日志里面说这两个对象有循环引用。

![图片](https://mmbiz.qpic.cn/mmbiz_png/deSLfic6WeGWMF8yPLzD5Rv3s26mceB6qecfIcBR92kk2rpQjj7EoHWNPWYz28iaZjUzLmK7S2EYju2OLcoAnWgw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)image-20210615183622686

`addMealToMealPlan` 这个函数看起来就有点可疑，让我们看一下。

![图片](https://mmbiz.qpic.cn/mmbiz_png/deSLfic6WeGWMF8yPLzD5Rv3s26mceB6qkbCmpoktCYaeEN6TEUHAaFG4qE1SwSqAR8uWsGCtoq0wp6j0ewCGibA/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)image-20210615183738662

这里我们调用了 addItem 到 mealPlan，也调用了 addPlan 到 menuItem。

![图片](https://mmbiz.qpic.cn/mmbiz_png/deSLfic6WeGWMF8yPLzD5Rv3s26mceB6qQH4FYmTib2xsL9PwNyeSDl582wheTcNDmKABMZArb5HpYLmpUZQorBQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)image-20210615184000286

果然，这里存在循环引用。MenuItem 持有 MealPlan， MealPlan 也会持有 MenuItem。这两个对象进行了互相的强引用。

当执行完 `addMealToMealPlan`，就没有任何引用指向 MenuItem 和 MealPlan 的对象了，但是他们依旧互相引用着对方，这就导致了泄露。

![图片](https://mmbiz.qpic.cn/mmbiz_png/deSLfic6WeGWMF8yPLzD5Rv3s26mceB6qBFhJFOa6jY9dACxSxOwPcIuhrfNYFkS4oxGOoticCuqUCCb6icJVhwUw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)image-20210615184444778

我们应该寻找一种方案来解决这个问题，这里使用了最快的解决方式，通过改变 MenuItem 对 MealPlan 的引用关系为弱引用来打破了引用循环。因为这里已经不存在强引用回环了。

### 检测和诊断堆分配回归

我们使用 `Meal Planner` 执行测试失败生成的 memgraph,检查一下堆内存增长的问题。

![图片](https://mmbiz.qpic.cn/mmbiz_png/deSLfic6WeGWMF8yPLzD5Rv3s26mceB6qz2rAl0nnz22gdS66xHH9YicbvzMnfBtrx9S7Y27cFHZfRGvl0jSBDIw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)image-20210615211510398

```
vmmap -summary app.memgraph
```

这里我们会对 pre 和 post memegraph 文件使用 vmmap 来获取内存使用的概览。

![图片](https://mmbiz.qpic.cn/mmbiz_png/deSLfic6WeGWMF8yPLzD5Rv3s26mceB6qPcbZyaKrJYU9PRGdYKIiceYRuiaWOO790HQekXlxs08ickrc3RFkfgfWA/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)image-20210615212406816

在 pre memegraph 中物理占用在 112 MB 左右。

![图片](https://mmbiz.qpic.cn/mmbiz_png/deSLfic6WeGWMF8yPLzD5Rv3s26mceB6qD9FV745KNW9ibJyQP6DTlOwB3xHDEUqecia9L60XWF1icnpgXyv4rreEQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)image-20210615212632183

而 post memegraph 中物理占用在 125 MB 左右。两次结果差值大概有 13 MB 左右。

![图片](https://mmbiz.qpic.cn/mmbiz_png/deSLfic6WeGWMF8yPLzD5Rv3s26mceB6qAxicGPoEK6qLsvia91PjzHj0GVF2JPA8rgbGveT81EWx6SFOkUQ5J4HA/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)image-20210615213215716

向下滚动输出，可以看见进程的内存占用被按 region 进行了划分。

![图片](https://mmbiz.qpic.cn/mmbiz_png/deSLfic6WeGWMF8yPLzD5Rv3s26mceB6q7pT8jY6YxyLUlgOEpzMQUmBdq8D6WVmsQpplGXWejng1BjEgbpB7ew/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)image-20210615213419470

因为我怀疑这块是一个堆分配的问题，所以我想从  MALLOC 范围的看看这些 regions。因为这些 regions 包含我所有的堆对象。

![图片](https://mmbiz.qpic.cn/mmbiz_png/deSLfic6WeGWMF8yPLzD5Rv3s26mceB6q66Dvm4llO84iaBtrNCXmkm3WV36U4T99OpUAUgJ1MYAPqEfzXWPlyJQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)image-20210615214206310

还记得上面说过的 **memory footprint = dirty size + compressed size** 在这个工具中 swapped 代表 compressed, 所以这些列我们只需要关心 **dirty size** 和 **swapped size** 。

![图片](https://mmbiz.qpic.cn/mmbiz_png/deSLfic6WeGWMF8yPLzD5Rv3s26mceB6qG5nVVjpLVuzibub4BDEyBblIg5EyMDc1Yy2jeicyoeEpYzmZL7YmXw6Q/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)image-20210615214254471

图上显示 MALLOC_LARGE 块大概持有 13 MB 的 dirty memory。这大概就等于我们回归的大小。所以我们需要查一下到底是谁贡献了这 13 MB。

![图片](https://mmbiz.qpic.cn/mmbiz_png/deSLfic6WeGWMF8yPLzD5Rv3s26mceB6qfsE7ctkM7QHkFRoRMkZTM5qfF89Lm9z6Vs4kd5FgANasibv1GPCMBcw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)image-20210615214621447

> 这里可以使用 `heap --diffFrom` 来看一下 pre 和 post 的差别。这个命令可以得到 post 中存在但 pre 中没有的对象。

最下面显示了这些 diff 大概有 13MB (13680384 bytes)。

![图片](https://mmbiz.qpic.cn/mmbiz_png/deSLfic6WeGWMF8yPLzD5Rv3s26mceB6qSaOVAspEUiafD2P5MxFRM9H4WJMDicVDnQP2FA5SickAnuzHJ9Jm5bfpw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)image-20210615215202648

高亮部分是按照类名来进行划分的。每一种对象都会有个数和大小的总结。这里我们可以看到我们大概有 13 MB 的 `non-object` 类型，在 Swift 中，这种类型通常是 raw 分配的bytes,这种对象可以用一点小技巧去追查。首要要拿到这些 `non-object` 的地址。

> 注意上图中 AVG 这一列代表的是每个 CLASS_NAME 对应 CLASS 对象的平均大小， `non-object` 类型对象的平均大小在 26777.3 byte，其他对象是没有超过 500 byte 的。

![图片](https://mmbiz.qpic.cn/mmbiz_png/deSLfic6WeGWMF8yPLzD5Rv3s26mceB6qPvWCanvVwxsZTB7C5DmHv6icGzfcRk6ckHI4RGibVNIRias5mIooqB28A/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)image-20210615215855078

可以使用如上 `heap -addresses` 命令来分析，特别注明只要 `non-object` 类型，并且大小至少 500 kb。如输出所示，`0x11380000` 地址的 `non-object` 大概有13 MB。所以它就是问题的根源。记录下来这个地址，我们需要通过这个地址来查询这些  `non-object` 的创建堆栈。

拿到对象地址后，我们有几个选择。可以根据具体情况来选择使用哪种方式继续追查，每种方法都有其好处,我将简要地逐一介绍。

**选择一**

```
 leaks --trace=address app.memgraph
```

这个命令可以得到这个地址的对象引用树，这个在查找特殊对象更多信息的时候很有用，特别是在 memgraph 没有打开 malloc stack logging 或者没有启用 MSL的时候 。注意 XCTest memgraph 会自动启用 MSL。

![图片](https://mmbiz.qpic.cn/mmbiz_png/deSLfic6WeGWMF8yPLzD5Rv3s26mceB6q6vWLKGo6uLTZC4r55jZLXKXqEd9hibOxQZ5roqv4LVzj5MHk5veS15w/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)image-20210616110444685

这个高亮的部分可能与我们的要查询的部分有关， 内存中的 MALLOC_LARGE 可能在 MKTCustomMealPlannerCollectionViewCell 中对 mealData 对象做了什么。

**选择二**

```
leaks --referenceTree app.memgraph
```

这个命令会得到一个进程中所有内存自上而下的引用树，他可以帮我们很好的推断出根节点。根据输出可以看出在 app 中内存聚集的场景，如果存在 large regression， 但是不知道是哪个对象引起的。用这个工具有奇效。

![图片](https://mmbiz.qpic.cn/mmbiz_png/deSLfic6WeGWMF8yPLzD5Rv3s26mceB6qRCc66FVnnybLRZWBrHv87BibKlYlB98sYf6GQibykgowoQdV794IJwxg/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)image-20210616114028220

我们可以传递 `--groupByType` 参数来把相同类型做一个聚类，来让输出更简洁，更容易看懂。large trunk regression 通常会在树中分组到一个节点下, 这让我们可以更容易发现那块内存是什么。注意上图高亮部分，同样是 mealData 大概有 13 MB。

![图片](https://mmbiz.qpic.cn/mmbiz_png/deSLfic6WeGWMF8yPLzD5Rv3s26mceB6qF0lZToc2BvUicA9O0dq4VK8Hfteqdt2Pia9QyjP5XXFeYqv8w7lEn0Rg/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)image-20210616120714875

因为 memgraph 启用了 MSL，所以可以使用 `malloc_hisatory -fullStacks` 来弄清楚这个对象怎么被创建的。

```
malloc_hisatory -fullStacks app.memgraph address
```

address 可以使用我们之前收集的地址，这样我们就可以得到这个地址的对象创建的堆栈了。请看上图第三行，  saveMeal 函数创建了这个对象。

**验证**

所有线索都指向 mealData， 现在让我们打开 Xcode 一探究竟。

![图片](https://mmbiz.qpic.cn/mmbiz_png/deSLfic6WeGWMF8yPLzD5Rv3s26mceB6qCBMOiazlhuFTBmG6YRz7ovXVQFMkS3SoeEC1ZMymrhHDN3X0VBgKVdw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)image-20210616164455729

找到罪魁祸首了，函数在一个自定义的 cell 的 view  里，这里创建了 raw buffer 并包装成了 mealData 对象。为了填充数据和保存数据到磁盘，这里创建了这个 buffer。一旦保存到磁盘，我们就不再需要这个 buffer 了。所以不应该一直用 mealData 属性保存着这个数据，因为只要这个 view 实例存在，这个 buffer 就会一直存在。这意味着，当我点击任意 cell 上的 saveMeal 按钮，这个 cell 就会创建并持有一个很大的 buffer，直到这个 cell 被销毁。当我点击多个 cell 上的保存按钮时，内存加起来就会很大。所以我们该如何解决？有两种常用的解决方式，可以根据具体情况进行选择。

**方法一**：我们只把 mealData 定义在函数中，但是我知道这个类的其他地方在使用 mealData，所以我不想这么做。

![图片](https://mmbiz.qpic.cn/mmbiz_png/deSLfic6WeGWMF8yPLzD5Rv3s26mceB6qRdQ4CbTLnArazrqK6LYicYsQ2Ly98KDzARSwKeTibBp6x6Cvic4gwUQJg/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)image-20210616170232527

**方法二**：另一个方法就是，当 mealData 保存到磁盘之后，手动置 nil。swift中的数据对象管理很聪明，一旦这个对象没有任何引用的时候，它就会被自动释放。

### 检测和诊断碎片化

![图片](https://mmbiz.qpic.cn/mmbiz_png/deSLfic6WeGWMF8yPLzD5Rv3s26mceB6q8ibdmV5bTqklzibkwx0vF9ZWGp1SJ5f5CPmfSbvzDnBQ5VR9covHlCrg/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)image-20210616223523732

我手动创建了一些对象，被标记为 my object，由于我没有太关注我的代码，系统最终交错安插了我的对象和其他对象，

![图片](https://mmbiz.qpic.cn/mmbiz_png/deSLfic6WeGWMF8yPLzD5Rv3s26mceB6qd5BhYVIwiaIIBV3XET2wdxCkoVrvSawaYNicxun50ibBcicBxOpeHrq3uA/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)image-20210616223633731

现在我释放掉了我的对象，出现了四个空闲的空槽，因为 allocated object 的存在，它们都没有连续，这将导致50%的碎片化和四个 dirty page。

![图片](https://mmbiz.qpic.cn/mmbiz_png/deSLfic6WeGWMF8yPLzD5Rv3s26mceB6qh0KeV1c0hsBXk2twmI2mc0NiaIZzgZkscXMQXUqasvDs9vFrt3iclepg/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)image-20210616224228526

假如我写的代码**一起创建了所有 my object**, 它们最终就会只会占用两个 page。

![图片](https://mmbiz.qpic.cn/mmbiz_png/deSLfic6WeGWMF8yPLzD5Rv3s26mceB6q1tTaXES4QpVcMtWpuCez85hmHHhAAP2ibPqHZYwyBicooEiapQu2GKPCw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)image-20210616224248384

当我释放掉所有的 my object，进程就会空出两个 clean page 给系统。结果就会得到两个 clean page 两个 dirty page 以及0%的碎片化。

注意碎片化是怎样成为占用空间的倍增器的。50%的碎片化就会让我们的内存占用翻倍。从两个 dirty page 变成 4 个 dirty page。

![图片](https://mmbiz.qpic.cn/mmbiz_png/deSLfic6WeGWMF8yPLzD5Rv3s26mceB6qcwj6pvwCWqDBAr1kMGcPEib8RFKEahrV9ichSPmP12PxRlvh7ZJkqRMg/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)image-20210617103651724

在大多数真实的场景中，一些碎片是不可避免的, 所以作为经验法则，把我们需要把**碎片化降低到25%或者更少**。

**使用 autorelease pool 是一种减少碎片的方式**，自动释放池会在执行超出释放池范围时告诉系统释放在它内部分配的所有对象,这有助于确保创建所有在释放池内的对象具有**相似的生命周期。**

尽管碎片化可能是所有进程的问题，**长时间运行的进程尤其容易产生碎片化**。因为他们有许多创建和销毁的对象，分割地址空间的可能性会更大。比如：如果你的 app 使用长时间运行 extensions，一定要看一看这些进程的碎片化。

![图片](https://mmbiz.qpic.cn/mmbiz_png/deSLfic6WeGWMF8yPLzD5Rv3s26mceB6qnleia6J20AZL7zsy27PDcw6FOPfrW2kdJLaKlZg963cia68Iqicbh0JzQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)image-20210617103956837

下面来快速的看一下我的进程碎片，我使用 `vmmap -sunmmary`,并且滚动到输出的最下面。

![图片](https://mmbiz.qpic.cn/mmbiz_png/deSLfic6WeGWMF8yPLzD5Rv3s26mceB6qNESM20AUf4icdGlwaxsmibkkceLswvbqq93R80cIcAmnicP0xqZZibSI8w/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)image-20210617104153630

高亮的部分按照 malloc zone 进行划分，每个 zone 包含不同类型的创建，通常我只需要关心 DefaultMallocZone，因为那是我的堆分配默认结束的地方。然而因为这个 memgraph 启用了 MSL，我真正关心的是 MallocStackLoggingLiteZone,只要启用了 MSL,这个区域就是所有堆分配结束的地方。

![图片](https://mmbiz.qpic.cn/mmbiz_png/deSLfic6WeGWMF8yPLzD5Rv3s26mceB6qSiczLTibIuwxLkd8eMDEtZT6gu2QoL5AayhSugrv7zTVfZnOwYvmDuuA/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)image-20210617105552030

`% FRAG` 这一列展示了我的内存在分配的所有 zone 上因为碎片产生浪费的百分比。他们中的一些数值真的比较大。但是我只需关心 `MallocStackLoggingLiteZone`，这个因为 `MallocStackLoggingLiteZone` 有最多的脏内存份额。脏内存总共5 MB，`MallocStackLoggingLiteZone` 占用4.3 MB。所以这种情况下，我可以忽略其他 zone。`dirty + swap frag size` 这一列精确的展示了因为碎片每一个 `malloc zone` 有多少内存被浪费了 。

![图片](https://mmbiz.qpic.cn/mmbiz_png/deSLfic6WeGWMF8yPLzD5Rv3s26mceB6qfI54keTib3xInDZEj6QkvNibRoyGSPibiabdOZz4icJztOFkX9XgIHLa8KA/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)image-20210617170856971

在这个 `case` 中，我因为碎片浪费了大约 800 KB。这个看起来很多，但是我们之前提过，一些碎片化问题是无法避免的，所以只要我还在 **25%** 碎片化以下，我就认为这个浪费是可以接受的。

![图片](https://mmbiz.qpic.cn/mmbiz_png/deSLfic6WeGWMF8yPLzD5Rv3s26mceB6qUVwuHrDDDq2n6YVMt4eeAfLwxdxJYbpnTOibibDgQCjTSbicT9GtvD8FA/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)image-20210617171420011

目前`MallocStackLoggingLiteZone` 的碎片化还在 19% 左右,这显然低于 25% 的经验法则,所以我还不用担心。

![图片](https://mmbiz.qpic.cn/mmbiz_png/deSLfic6WeGWMF8yPLzD5Rv3s26mceB6qUuVkicM9TCF0MUfvPGAxMDaxGHwdZYZWPn3JDYzRWVXDoXgEdLVSUpQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)image-20210617195647203

如果我真的有碎片化问题，我可以使用 `instruments` 的工具 `Allocations` 去追踪这个问题，具体来说，我希望查看分配列表视图，看看在我感兴趣的区域中哪些对象被持久化和销毁了。

![图片](https://mmbiz.qpic.cn/mmbiz_png/deSLfic6WeGWMF8yPLzD5Rv3s26mceB6qSO1liaa3kqJc69zdGTEbv5TrokegTibXbySIYKvlnfIsr5bLXXI0HN2A/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)image-20210617195714870

在碎片化的背景下，被销毁的对象创建了内存空槽，而持久化对象是剩余的对象负责保持 `dirty page`, 当你研究碎片化的时候，它们都值得研究。想知道怎样使用 `instruments`工具的更多信息请参阅：**WWDC19 Getting Started with Instruments**[17]

![图片](https://mmbiz.qpic.cn/mmbiz_png/deSLfic6WeGWMF8yPLzD5Rv3s26mceB6q3CFC1Dltvpx5zowOHdoFiaxzLL8LGLMJEMsvTSqPW9uibAjbkzMJCCzQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)image-20210617200756322

总结一下如何解决碎片化问题：

- 尽量保证连续创建生命周期相似的对象
- 碎片化尽量降低到 25% 或者更少
- 使用 autorelease pool 是一种减少碎片的方式
- 长时间运行的进程尤其容易产生碎片化，多关注一下这些进程的碎片化。
- 也可以使用 instruments 的 allocations 工具来诊断碎片化问题。

### 回顾

![图片](https://mmbiz.qpic.cn/mmbiz_png/deSLfic6WeGWMF8yPLzD5Rv3s26mceB6qW9Qyax7eItmqpkE2tQzpxhdmPh0SbqFtrY1CrmLSDkVSibCJUl9tXxw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)image-20210617200949635

现在，我已经解决了泄露和堆回归，验证了碎片化不是一个问题。在此运行 `Xcode`。太棒了，现在测试通过了，并且回归问题也被解决。现在你已经学习到了关于 检测和诊断内存问题，让我们来回顾一下在你自己 app 可以使用哪些工作流程。

#### 检测流程

![图片](https://mmbiz.qpic.cn/mmbiz_png/deSLfic6WeGWMF8yPLzD5Rv3s26mceB6qhUsficZA0xs9XxlkiaND6x6bWAjtvrPhYxrCEILzicSsMYEdPSDVyEypw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)image-20210617201641291

开发一个新功能之后，用 `XCTest` 写一个性能测试来监测内存,和其他系统指标。为每个测试设置 baseline。然后用测试来捕获回归，并使用收集到的 `ktrace` 和 `memgraph` 文件进行调查。

#### 诊断流程

![图片](https://mmbiz.qpic.cn/mmbiz_png/deSLfic6WeGWMF8yPLzD5Rv3s26mceB6qd7hYq8BmvqK76j6dInHEXU7uib90S4g5OxKN3iaPuxZj0bYn61Drk8TQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)image-20210617202711003

使用在执行 `XCTest` 失败生成的 memgraph 文件来帮忙诊断你的内存问题，首先你应该检查泄露，使用 `leaks` 工具并且使用 MSL 堆栈来帮忙找到需要修复的泄露。如果回归不包含泄露，再去检查堆。使用 `vmmap -summary` 来确认堆上的内存。如果需要,使用 `heap -diffFrom` 来查看那个对象类型造成了内存增长。如果某个对象类型很可疑, 使用 `heap -addresses` 来获取地址。如果罪魁祸首看起来并不明显，尝试使用 `leaks -referenceTree` 来找到一些线索。并且搭配 `leaks -traceTree` `malloc_history` 来找到有问题的对象地址。

最后，请确保把这些最佳实践牢记在心。努力让你的应用程序零泄漏。

**总结**：

1. 如果你使用 `unsafe` 类型，**确保你会释它**。
2. 同时也要注意代码中的循环引。
3. 找到一种方式来减少你的堆分配，你可以缩小它们, 并且尽量把持有它们的时间变的更短。或者完全**取消不必要的分配**。
4. 确保把碎片化问题牢记在心，创建的对象要尽量相邻并且具有相似的生命周期。
5. 使用这些最佳实践和 `XCTest` 工作流，您将能够检测、诊断和修复应用程序中的内存问题。

**作者总结**：

本 session 主要介绍如何使用 XCTest 写性能测试来检测内存问题(泄露和碎片化)，这是一种可重用的，更加系统性的检测内存性能的方式，因为是通过命令行工具对文件进行分析，你可以通过脚本快速检测泄露和堆的问题，简化一些无用信息的输出。但是每个 app 都有自己的情况，可能因为种种原因，目前还不能使用 XCTest 来对 app 进行测试，或者开发者目前对内存问题查找及命令工具不熟悉的时候，Xcode  的 `Debug Memory Graph`就是一个很好的入门工具。它可以很方便的帮我们查找泄露，并且这个工具是可视化的。你只需要运行工程，用手点一遍自己的新功能，然后点击 `Debug Memory Graph` 来捕获内存图，通过筛选来看内存中的泄露，或者查看目前的存活对象及其创建堆栈和引用关系， `Debug Memory Graph` 是一种轻量级的，更方便、更快速、更直观的方式来让你了解自己 app 内存的使用情况。如果需要，你还可以在 `Debug Memory Graph` 时，导出当前捕获的内存图的 memgraph 文件，可以多次导出然后就可以使用命令行工具 heap 新增的 diffFrom 功能了哦，你可以结合上面学到的命令工具，帮你最大程度的了解内存问题。想要更详细的了解  `Debug Memory Graph` ，建议观看 **WWDC18 iOS Memory Deep Dive**[18] 或者阅读 **深入解析iOS内存 iOS Memory Deep Dive**[19] 来获取更多信息。