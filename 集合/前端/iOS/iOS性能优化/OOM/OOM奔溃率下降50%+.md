# [iOS性能优化实践：头条抖音如何实现OOM崩溃率下降50%+](https://mp.weixin.qq.com/s?__biz=MzI1MzYzMjE0MQ==&mid=2247486858&idx=1&sn=ec5964b0248b3526836712b26ef1b077&chksm=e9d0c668dea74f7e1e16cd5d65d1436c28c18e80e32bbf9703771bd4e0563f64723294ba1324&scene=178&cur_album_id=1568330323321470981#rd)



> iOS OOM 崩溃在生产环境中的归因一直是困扰业界已久的疑难问题，字节跳动旗下的头条、抖音等产品也面临同样的问题。
>
> 在字节跳动性能与稳定性保障团队的研发实践中，我们自研了一款基于内存快照技术并且可应用于生产环境中的 OOM 归因方案——线上 Memory Graph。基于此方案，3 个月内头条抖音 OOM 崩溃率下降 50%+。
>
> 本文主要分享下该解决方案的技术背景，技术原理以及使用方式，旨在为这个疑难问题提供一种新的解决思路。

# OOM 崩溃背景介绍

## OOM

OOM 其实是`Out Of Memory`的简称，指的是在 iOS 设备上当前应用因为内存占用过高而被操作系统强制终止，在用户侧的感知就是 App 一瞬间的闪退，与普通的 Crash 没有明显差异。但是当我们在调试阶段遇到这种崩溃的时候，从设备`设置->隐私->分析与改进`中是找不到普通类型的崩溃日志，只能够找到`Jetsam`开头的日志，这种形式的日志其实就是 OOM 崩溃之后系统生成的一种专门反映内存异常问题的日志。那么下一个问题就来了，什么是`Jetsam`？

## Jetsam

`Jetsam`是 iOS 操作系统为了控制内存资源过度使用而采用的一种资源管控机制。不同于`MacOS`，`Linux`，`Windows`等桌面操作系统，出于性能方面的考虑，iOS 系统并没有设计内存交换空间的机制，所以在 iOS 中，如果设备整体内存紧张的话，系统只能将一些优先级不高或占用内存过大的进程直接终止掉。

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/5EcwYhllQOjsdyfHiajeANfS7VBvxfZiaRSt2hNkIsXSvShh9fgYHX5L7LL8lHx1J1OwC59Mica7Jgog3bz9mUE9w/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

Jetsam 日志解读

上图是截取一份`Jetsam`日志中最关键的一部分。关键信息解读：

- pageSize:指的是当前设备物理内存页的大小，当前设备是`iPhoneXs Max`，大小是 16KB，苹果 A7 芯片之前的设备物理内存页大小则是 4KB。
- states:当前应用的运行状态，对于`Heimdallr-Example`这个应用而言是正在前台运行的状态，这类崩溃我们称之为`FOOM`(Foreground Out Of Memory)；与此相对应的也有应用程序在后台发生的 OOM 崩溃，这类崩溃我们称之为`BOOM`(Background Out Of Memory)。
- rpages:是`resident pages`的缩写，表明进程当前占用的内存页数量，Heimdallr-Example 这个应用占用的内存页数量是 92800，基于 pageSize 和 rpages 可以计算出应用崩溃时占用的内存大小:16384 * 92800 / 1024 /1024 = 1.4GB。
- reason:表明进程被终止的的原因，`Heimdallr-Example`这个应用被终止的原因是超过了操作系统允许的单个进程物理内存占用的上限。

`Jetsam`机制清理策略可以总结为下面两点：

\1. 单个 App 物理内存占用超过上限

\2.  整个设备物理内存占用收到压力按照下面优先级完成清理：

1. 1. 后台应用>前台应用
   2. 内存占用高的应用>内存占用低的应用
   3. 用户应用>系统应用

`Jetsam`的代码在开源的`XNU`代码中可以找到，这里篇幅原因就不具体展开了，具体的源码解析可以参考本文最后第 2 和第 3 篇参考文献。

## 为什么要监控 OOM 崩溃

前面我们已经了解到，OOM 分为`FOOM`和`BOOM`两种类型，显然前者因为用户的感知更明显，所以对用户的体验的伤害更大，下文中提到的 OOM 崩溃仅指的是`FOOM`。那么针对 OOM 崩溃问题有必要建立线上的监控手段吗？

答案是有而且非常有必要的！原因如下：

1. 重度用户也就是使用时间更长的用户更容易发生`FOOM`，对这部分用户体验的伤害导致用户流失的话对业务损失更大。
2. 头条，抖音等多个产品线上数据均显示`FOOM`量级比普通崩溃还要多，因为过去缺乏有效的监控和治理手段导致问题被长期忽视。
3. 内存占用过高即使没导致`FOOM`也可能会导致其他应用`BOOM`的概率变大，一旦用户发现从微信切换到我们 App 使用，再切回微信没有停留在之前微信的聊天页面而是重新启动的话，对用户来说，体验是非常糟糕的。

# OOM 线上监控

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/5EcwYhllQOjsdyfHiajeANfS7VBvxfZiaRcibd6j3tT74mB5HkMIBXJ4rJXFnibTFHnh43o1cEeXBpU5GZT0UE1X3Q/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

Jetsam 强杀代码截图

翻阅`XNU`源码的时候我们可以看到在`Jetsam`机制终止进程的时候最终是通过发送`SIGKILL`异常信号来完成的。

> \#define SIGKILL 9 kill (cannot be caught or ignored)

从系统库 signal.h 文件中我们可以找到`SIGKILL`这个异常信号的解释，它不可以在当前进程被忽略或者被捕获，我们之前监听异常信号的常规 Crash 捕获方案肯定也就不适用了。那我们应该如何监控 OOM 崩溃呢？

正面监控这条路行不通，2015 年的时候`Facebook`提出了另外一种思路，简而言之就是排除法。具体流程可以参考下面这张流程图：

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/5EcwYhllQOjsdyfHiajeANfS7VBvxfZiaRKw1VwCUKcLHgo5ZmK6lKQbOzfnuicD4hQhqnugkzA2r7lU4P5BtadGA/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

排除法判定OOM崩溃的流程

我们在每次 App 启动的时候判断上一次启动进程终止的原因，那么已知的原因有：

- App 更新了版本
- App 发生了崩溃
- 用户手动退出
- 操作系统更新了版本
- App 切换到后台之后进程终止

如果上一次启动进程终止的原因不是上述任何一个已知原因的话，就判定上次启动发生了一次`FOOM`崩溃。

曾经`Facebook`旗下的`Fabric`也是这样实现的。但是通过我们的测试和验证，上述这种方式至少将以下几种场景误判：

- WatchDog 崩溃
- 后台启动
- XCTest/UITest 等自动化测试框架驱动
- 应用 exit 主动退出

在字节跳动 OOM 崩溃监控上线之前，我们已经排除了上面已知的所有误判场景。需要说明的是，因为排除法毕竟没有直接的监控来的那么精准，或多或少总有一些 bad case，但是我们会保证尽量的准确。

# 自研线上 Memory Graph，OOM 崩溃率下降 50%+

## OOM 生产环境归因

目前在 iOS 端排查内存问题的工具主要包括 Xcode 提供的 Memory Graph 和 Instruments 相关的工具集，它们能够提供相对完备的内存信息，但是应用场景仅限于开发环境，无法在生产环境使用。由于内存问题往往发生在一些极端的使用场景，线下开发测试一般无法覆盖对应的问题，Xcode 提供的工具无法分析处理大多数偶现的疑难问题。

对此，各大公司都提出了自己的线上解决方案，并开源了例如`MLeaksFinder`、`OOMDetector`、`FBRetainCycleDetector`等优秀的解决方案。

在字节跳动内部的使用过程中，我们发现现有工具各有侧重，无法完全满足我们的需求。主要的问题集中在以下两点：

- 基于 Objective-C 对象引用关系找循环引用的方案，适用范围比较小，只能处理部分循环引用问题，而内存问题通常是复杂的，类似于内存堆积，Root Leak，C/C++层问题都无法解决。
- 基于分配堆栈信息聚类的方案需要常驻运行，对内存、CPU 等资源存在较大消耗，无法针对有内存问题的用户进行监控，只能广撒网，用户体验影响较大。同时，通过某些比较通用的堆栈分配的内存无法定位出实际的内存使用场景，对于循环引用等常见泄漏也无法分析。

为了解决头条，抖音等各产品日益严峻的内存问题，我们自行研发了一款基于内存快照技术的线上方案，我们称之为——线上 Memory Graph。上线后接入了集团内几乎所有的产品，帮助各产品修复了多年的历史问题，OOM 率降低一个数量级，3 个月之内抖音最新版本 OOM 率下降了 50%，头条下降了 60%。线上突发 OOM 问题定位效率大大提升，彻底告别了线上 OOM 问题归因“两眼一抹黑”的时代。

线上 Memory Graph 核心的原理是扫描进程中所有 Dirty 内存，通过内存节点中保存的其他内存节点的地址值建立起内存节点之间的引用关系的有向图，用于内存问题的分析定位，整个过程不使用任何私有 API。这套方案具备的能力如下：

1. 完整还原用户当时的内存状态。
2. 量化线上用户的大内存占用和内存泄漏，可以精确的回答 App 内存到底大在哪里这个问题。
3. 通过内存节点符号和引用关系图回答内存节点为什么存活这个问题。
4. 严格控制性能损耗，只有当内存占用超过异常阈值的时候才会触发分析。没有运行时开销，只有采集时开销，对 99.9%正常使用的用户几乎没有任何影响。
5. 支持主要的编程语言，包括 OC，C/C++，Swift，Rust 等。

线上 Memory Graph 采集及上报流程示意图

![图片](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)

## 内存快照采集 

线上 Memory Graph 采集内存快照主要是为了获取当前运行状态下所有内存对象以及对象之间的引用关系，用于后续的问题分析。主要需要获取的信息如下：

- 所有内存的节点，以及其符号信息（如`OC/Swift/C++` 实例类名，或者是某种有特殊用途的 VM 节点的 tag 等）。
- 节点之间的引用关系，以及符号信息（偏移，或者实例变量名），`OC/Swift`成员变量还需要记录引用类型。

由于采集的过程发生在程序正常运行的过程中，为了保证不会因为采集内存快照导致程序运行异常，整个采集过程需要在一个相对静止的运行环境下完成。因此，整个快照采集的过程大致分为以下几个步骤：

1. 挂起所有非采集线程。
2. 获取所有的内存节点，内存对象引用关系以及相应的辅助信息。
3. 写入文件。
4. 恢复线程状态。

下面会分别介绍整个采集过程中一些实现细节上的考量以及收集信息的取舍。

### 内存节点的获取

程序的内存都是由虚拟内存组成的，每一块单独的虚拟内存被称之为`VM Region`，通过 mach 内核的`vm_region_recurse/vm_region_recurse64`函数我们可以遍历进程内所有`VM Region`，并通过`vm_region_submap_info_64`结构体获取以下信息：

- 虚拟地址空间中的地址和大小。
- Dirty 和 Swapped 内存页数，表示该`VM Region`的真实物理内存使用。
- 是否可交换，Text 段、共享 mmap 等只读或随时可以被交换出去的内存，无需关注。
- user_tag，用户标签，用于提供该`VM Region`的用途的更准确信息。

大多数 VM Region 作为一个单独的内存节点，仅记录起始地址和 Dirty、Swapped 内存作为大小，以及与其他节点之间的引用关系；而 libmalloc 维护的堆内存所在的 VM Region 则由于往往包含大多数业务逻辑中的 Objective-C 对象、C/C++对象、buffer 等，可以获取更详细的引用信息，因此需要单独处理其内部节点、引用关系。

在 iOS 系统中为了避免所有的内存分配都使用系统调用产生性能问题，相关的库负责一次申请大块内存，再在其之上进行二次分配并进行管理，提供给小块需要动态分配的内存对象使用，称之为堆内存。程序中使用到绝大多数的动态内存都通过堆进行管理，在 iOS 操作系统上，主要的业务逻辑分配的内存都通过`libmalloc`进行管理，部分系统库为了性能也会使用自己的单独的堆管理，例如`WebKit`内核使用`bmalloc`，`CFNetwork`也使用自己独立的堆，在这里我们只关注`libmalloc`内部的内存管理状态，而不关心其它可能的堆（即这部分特殊内存会以`VM Region`的粒度存在，不分析其内部的节点引用关系）。

我们可以通过`malloc_get_all_zones`获取`libmalloc`内部所有的`zone`，并遍历每个`zone`中管理的内存节点，获取 libmalloc 管理的存活的所有内存节点的指针和大小。

### 符号化

获取所有内存节点之后，我们需要为每个节点找到更加详细的类型名称，用于后续的分析。其中，对于 VM Region 内存节点，我们可以通过 user_tag 赋予它有意义的符号信息；而堆内存对象包含 raw buffer，Objective-C/Swift、C++等对象。对于 Objective-C/Swift、C++这部分，我们通过内存中的一些运行时信息，尝试符号化获取更加详细的信息。

Objective/Swift 对象的符号化相对比较简单，很多三方库都有类似实现，`Swift`在内存布局上兼容了`Objective-C`，也有`isa`指针，`objc`相关方法可以作用于两种语言的对象上。只要保证 isa 指针合法，对象实例大小满足条件即可认为正确。

C++对象根据是否包含虚表可以分成两类。对于不包含虚表的对象，因为缺乏运行时数据，无法进行处理。

对于对于包含虚表的对象，在调研 mach-o 和 C++的 ABI 文档后，可以通过 std::type_info 和以下几个 section 的信息获取对应的类型信息。

- `type_name string` - 类名对应的常量字符串，存储在`__TEXT/__RODATA`段的`__const section`中。
- `type_info` - 存放在`__DATA/__DATA_CONST`段的`__const section`中。
- `vtable` - 存放在`__DATA/__DATA_CONST`段的`__const section`中。

![图片](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)

C++实例以及 vtable 的引用关系示意图

在 iOS 系统内，还有一类特殊的对象，即`CoreFoundation`。除了我们熟知的`CFString`、`CFDictionary`外等，很多很多系统库也使用 CF 对象，比如`CGImage`、`CVObject`等。从它们的 isa 指针获取的`Objective-C`类型被统一成`__NSCFType`。由于 CoreFoundation 类型支持实时的注册、注销类型，为了细化这部分的类型，我们通过逆向拿到 CoreFoundation 维护的类型 slot 数组的位置并读取其数据，保证能够安全的获取准确的类型。

![图片](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)CoreFoundation 类型获取

### 引用关系的构建

整个内存快照的核心在于重新构建内存节点之间的引用关系。在虚拟内存中，如果一个内存节点引用了其它内存节点，则对应的内存地址中会存储指向对方的指针值。基于这个事实我们设计了以下方案：

1. 遍历一个内存节点中所有可能存储了指针的范围获取其存储的值 A。
2. 搜索所有获得的节点，判断 A 是不是某一个内存节点中任何一个字节的地址，如果是，则认为是一个引用关系。
3. 对所有内存节点重复以上操作。



对于一些特定的内存区域，为了获取更详细的信息用于排查问题，我们对栈内存以及 Objective-C/Swift 的堆内存进行了一些额外的处理。

其中，栈内存也以`VM Region`的形式存在，栈上保存了临时变量和 TLS 等数据，获取相应的引用信息可以帮助排查诸如 autoreleasepool 造成的内存问题。由于栈并不会使用整个栈内存，为了获取 Stack 的引用关系，我们根据寄存器以及栈内存获取当前的栈可用范围，排除未使用的栈内存造成的无效引用。

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/5EcwYhllQOjsdyfHiajeANfS7VBvxfZiaRpSkXYH1hiajJREAjv73hKDTx1Pj95CmmzlBTKA1ibib5QJJJqkib8NumFA/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

栈使用范围

而对于`Objective-C/Swift`对象，由于运行时包含额外的信息，我们可以获得`Ivar`的强弱引用关系以及`Ivar`的名字，带上这些信息有助于我们分析问题。通过获得`Ivar`的偏移，如果找到的引用关系的偏移和`Ivar`的偏移一致，则认为这个引用关系就是这个`Ivar`，可以将`Ivar`相关的信息附加上去。

## 数据上报策略

我们在 App 内存到达设定值后采集 App 当时的内存节点和引用关系，然后上传至远端进行分析，可以精准的反映 App 当时的内存状态，从而定位问题，总的流程如下：

线上 Memory Graph 整体工作流程

![图片](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)

整个线上 Memory Graph 模块工作的完整流程如上图所示，主要包括：

1. 后台线程定时检测内存占用，超过设定的危险阈值后触发内存分析。
2. 内存分析后数据持久化，等待下次上报。
3. 原始文件压缩打包。
4. 检查后端上报许可，因为单个文件很大，后端可能会做一些限流的策略。
5. 上报到后端分析，如果成功后清除文件，失败后会重试，最多三次之后清除，防止占用用户太多的磁盘空间。

## 后台分析

这是字节监控平台 Memory Graph 单点详情页的一个 case：

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/5EcwYhllQOjsdyfHiajeANfS7VBvxfZiaR50KtM5ANgoYyyo4ZGJHfcZwLcXVNicfAPuCm24es4icIA0D0QFuObRng/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

线上 Memory Graph 详情页概览

我们可以看到这个用户的内存占用已经将近 900MB，我们分析时候的思路一般是：

1. 从对象数量和对象内存占用这两个角度尝试找到类列表中最有嫌疑的那个类。
2. 从对象列表中随机选中某个实例，向它的父节点回溯引用关系，找到你认为最有嫌疑的一条引用路径。
3. 点击引用路径模块右上角的`Add Tag`来判断当前选中的引用路径在同类对象中出现过多少次。
4. 确认有问题的引用路径之后再判断究竟是哪个业务模块发生的问题。

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/5EcwYhllQOjsdyfHiajeANfS7VBvxfZiaRAAicRastryljXlnFLrVvKlDoo7MNh8BWFhWgv7jiaYiaicr9QqqXJre0tQ/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

当前引用路径在同类型对象中出现频率统计

通过上图中引用路径的分析我们发现，所有的图片最终都被`TTImagePickController`这个类持有，最终排查到是图片选择器模块一次性把用户相册中的所有图片都加载到内存里，极端情况下会发生这个问题。

## 整体性能和稳定性

### 采集侧优化策略

由于整个内存空间一般包含的内存节点从几十万到几千万不等，同时程序的运行状态瞬息万变，采集过程有着很大的性能和稳定性的压力。

我们在前面的基础上还进行了一些性能优化：

- 写出采集数据使用`mmap`映射，并自定义二进制格式保证顺序读写。
- 提前对内存节点进行排序，建立边引用关系时使用二分查找。通过位运算对一些非法内存地址进行提前快速剪枝。

对于稳定性部分，我们着重考虑了下面几点：

- 死锁

由于无法保证 Objective-C 运行时锁的状态，我们将需要通过运行时 api 获取的信息在挂起线程前提前缓存。同时，为了保证`libmalloc`锁的状态安全，在挂起线程后我们对 libmalloc 的锁状态进行了判断，如果已经锁住则恢复线程重新尝试挂起，避免堆死锁。

- 非法内存访问

在挂起所有其他线程后，为了减少采集本身分配的内存对采集的影响，我们使用了一个单独的`malloc_zone`管理采集模块的内存使用。

### 性能损耗

因为在数据采集的时候需要挂起所有线程，会导致用户感知到卡顿，所以字节模块还是有一定性能损耗的，经过我们测试，在`iPhone8 Plus`设备上，App 占用 1G 内存时，采集用时 1.5-2 秒，采集时额外内存消耗 10-20MB，生成的文件 zip 后大小在 5-20MB。

为了严格控制性能损耗，线上 Memory Graph 模块会应用以下策略，避免太频繁的触发打扰用户正常使用，避免自身内存和磁盘等资源过多的占用：

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/5EcwYhllQOjsdyfHiajeANfS7VBvxfZiaRkcmVTPu4P1hmDu8C1oicqODmPoUZWgvHCjpYX9IECdGhJthepZu0X2Q/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

性能损耗控制策略

### 稳定性

该方案已经在字节全系产品线上稳定运行了 6 个月以上，稳定性和成功率得到了验证，目前单次采集成功率可以达到 99.5%，剩下的失败基本都是由于内存紧张提前 OOM，考虑到大多数应用只有不到千分之一的用户会触发采集，这种情况属于极低概率事件。

## 试用路径

目前，线上 Memory Graph 已搭载在字节跳动火山引擎旗下应用性能管理平台（APMInsight）上赋能给外部开发者使用。

APMInsight 的相关技术经过今日头条、抖音、西瓜视频等众多应用的打磨，已沉淀出一套完整的解决方案，能够定位移动端、浏览器、小程序等多端问题，除了支持崩溃、错误、卡顿、网络等基础问题的分析，还提供关联到应用启动、页面浏览、内存优化的众多功能。目前 Demo 已开放大部分能力，欢迎各位注册账号试用：https://www.volcengine.cn/product/apminsight