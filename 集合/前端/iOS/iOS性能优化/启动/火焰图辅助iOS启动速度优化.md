# 火焰图辅助iOS启动优

# 一、背景

距离上次启动优化（启动任务分级）相隔差不多2年时间了,虽然一直保持在之前的启动速度，但是每个版本排查启动增量会耗费不少时间,想做一个自动化的启动监控流程来降低这方面的时间成本，在启动监控开发中又发现部分启动可优化，于是就顺便把启动也优化了一下。

本文主要涉及以下几方面：

- **1、启动优化**：启动流程、如何优化、push启动优化、二进制重排、后续计划
- **2、自动化启动监控**

# 二、成果

**1、启动优化**：在iPhone8Plus上自测，从点击图标到首页图片完全加载由之前的1.2s减少到0.51s。测试同学分别在iPhone6和iPhone8上面验证，总启动耗时相比线上版本减少了 50%-60% 。

**2、启动监控**：每晚固定的时间点，设备会自动启动应用10次，将启动数据上传并diff上一天的数据，将diff数据增量超标的方法通过邮件发送到代码提交者的邮箱，提示对应同学修改。

下图为8plus优化后的启动

![aaa1.webp](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/14c75c7d0c1247eabafa8847a587ea3d~tplv-k3u1fbpfcp-watermark.awebp)

# 三、优化思路

## 1、如何定义启动开始和结束时间？

在做优化之前，需要将启动耗时的计算标准规范统一化，这样才好衡量启动耗时以及优化的效果。

### 1.1 启动流程

根据下图，定义出启动开始时间为用户点击icon，结束时间为首页数据展示完成

![p2.webp](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/01bffabfedfa47c3be70126f7ee08db0~tplv-k3u1fbpfcp-watermark.awebp)

### 1.2 计算启动开始和结束时间

#### 1.2.1 测试标准：

使用录屏工具对app启动进行录制，通过QuickTime Plyaer的修剪功或将者视频解帧计算，以点击appicon变灰为启动开始时间、以首页图片完全展示为结束时间，计算两个时间的差即为总启动时间。

#### 1.2.2 代码如何统计：

- **启动时间**：通过当前进程标识（NSProcessInfo\processIdentifier），读取进程信息内的进程创建时间(__p_starttime)为启动时间。

```
+ (NSTimeInterval)processStartTime
{   // 单位是毫秒
    struct kinfo_proc kProcInfo;
    if ([self processInfoForPID:[[NSProcessInfo processInfo] processIdentifier] procInfo:&kProcInfo]) {
        return kProcInfo.kp_proc.p_un.__p_starttime.tv_sec * 1000.0 + kProcInfo.kp_proc.p_un.__p_starttime.tv_usec / 1000.0;
        
    } else {
        NSAssert(NO, @"无法取得进程的信息");
        return 0;
    }
}

+ (BOOL)processInfoForPID:(int)pid procInfo:(struct kinfo_proc*)procInfo
{
    int cmd[4] = {CTL_KERN, KERN_PROC, KERN_PROC_PID, pid};
    size_t size = sizeof(*procInfo);
    return sysctl(cmd, sizeof(cmd)/sizeof(*cmd), procInfo, &size, NULL, 0) == 0;
}
复制代码
```

- **结束时间**：以首页的所有图片全部加载完成为结束时间，hook图片下载方法，在启动完成前将所有调用该方法的url存入数组，图片下载完成之后移出数组，当数组内元素个数为0时，代表首页的图片下载完成，即为结束时间，以下为hook的伪代码：

```
- (void)hook_setImageWithUrl:(NSString *)url completed:(completedBlock)completed
{
    // 启动已经完成执行hook前逻辑
    if (LaunchSteps.launchFinished) {
        [self hook_setimageWithUrl...];
        return;
    }
    [LaunchImageArray addobject:url];
    completedBlock newCompletedBlock = ^(...) {
        [LaunchImageArray removeObject:url];
        if (LaunchImageArray.count == 0) { // 数组个数为0代表全部图片下载完成
            LaunchSteps.launchFinished = YES;
        }
        if (completed) {
            completed(...);
        }
    }
    [self hook_setImageWithUrl:url completed:newCompletedBlock];
    
}
复制代码
```

## 2、优化两步骤

### 2.1 找

根据启动流程，找出app启动时耗时较大的、启动流程中不需要的方法。

### 2.2 改

对耗时较高的方法，进行耗时细分，寻找可优化部分，进行修改。对启动流程中不需要的方法，进行懒加载或者延后到启动完成之后执行。

## 3、pre-main优化

pre-main的整个流程可以看前面的图，已经有非常成熟且多的资料对这个流程进行了说明，这里就不重复了，总之这个阶段我们能做的有：

- 1、Load dylibs阶段：减少或者合并dylibs，将动态库换成静态库。
- 2、Rebase/Bind阶段：减少类、方法、分类数量
- 3、Objc setup阶段：没啥可做的
- 4、Initializers阶段：优化+load方法、减少构造器函数（constructor），减少C++静态全局变量

以上部分，其实在上一次启动优化（两年前）就已经做的差不多了（没有处理过的建议先处理一下），比如公司内部的sdk已经全部换成静态库了，category、load方法也处理过，删除无用类、方法、资源这个在很早以前做包体积优化的时候已经做得比较彻底了，并且现在也有一套自动化流程来管理每天包体积的增量，所以整个pre-main的启动优化能做的非常少，不过为了突破原有的优化过的速度，也做了一些苦力活，本身因为项目历史悠久，并且代码数量较大库较多，导致pre-main的整个耗时比较高，于是想衡量一下每个库的引入对整个项目的启动造成了多大的影响，通过创建一个新的工程，分别将podfile里面的库一个个的导入进新项目，然后大概的评估每个库带来的pre-main耗时，步骤就是：

首先在xcode设置环境变量 DYLD_PRINT_STATISTICS 为1，这个能输出pre-main的耗时，然后。

- 1、podfile 添加 podA
- 2、pod update
- 3、重启设备（重要！），xcode运行新项目，记录pre-main耗时，比较未添加podA库时的耗时差值，然后重复1、2、3步骤，大概统计出每个库引入项目带来的pre-main耗时影响。

得出每个库大概的耗时之后，我们评估出有一些库并不那么重要并且耗时达到几十毫秒的例如某Refresh库等（本身有一套类似逻辑），我们将它移除并且修改使用的部分。对耗时较高方便推动修改的库推动优化（不方便推动的去提需求容易被打，注意安全 ⚠️）这一步大概移除了3-5个库。

## 4、main阶段优化

main阶段的优化第一步找出可优化的任务，提供三种找的方式：

#### 方案一：

通过走查代码，看哪些任务在整个启动链路上是不必要的,进行延后，使用插桩打点的方式通过NSLog输出每个方法执行后的时间统计每个方法的耗时，对耗时高的进行耗时细分，可拆解的进行拆解，可延后的进行延后。这个方案比较直接和简单，如果没有进行任务的优先级排序，这个方式也能加快启动速度，缺点就是无法找出一些依赖关系导致的一些不必要的任务执行。

```
// 通过这个方式来统计每个方法同步的耗时
CFAbsoluteTime start = CFAbsoluteTimeGetCurrent();
[self doSomething];
NSLog(@"doSomething : %f",CFAbsoluteTimeGetCurrent() - start);

// 可以在main函数调用的时候设置一个全局开始时间，
// 在其他类里面通过extern关键字取main的时间，如在main.m内:
CFAbsoluteTime kAppStartTime;
int main(int argc, char *argv[])
{
    kAppStartTime = CFAbsoluteTimeGetCurrent();
}
// someClass.m
extern CFAbsoluteTime kAppStartTime;
CFAbsoluteTime duration = (CFAbsoluteTimeGetCurrent() - kAppStartTime);
复制代码
```

#### 方案二：

通过hook objc_msgSend方法，统计main-->首页图片完全加载的所有方法以及耗时，按照火焰图需要的数据格式生成一个json文件，将该json文件传入分析工具[chrome://tracing/](https://link.juejin.cn?target=undefined)生成火焰图，通过以下火焰图，我们可以非常方便的看到启动时执行了哪些方法和耗时的多少，接下来需要分析每个任务在启动时调用的必要性然后再针对其进行优化，以下为未优化时的火焰图：

![p3.webp](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/1711d51bdf90476185fba6b97961a1de~tplv-k3u1fbpfcp-watermark.awebp)

从左到右为启动时间轴，从上到下为：方法A里面调用了方法B、C、D。方法A就在最上层，BCD就在下一层，例如APPDelegate的swizzied_didFinishLuanch方法里面调用了launch...和MainTabbarController。以此类推可以找到最终调用到了哪个方法导致的耗时。

#### 方案三：APP Launch工具

APP Launch工具是目前来说启动优化最强最全面的检测工具并且它也是苹果官方推荐的[官方地址](https://link.juejin.cn?target=https%3A%2F%2Fdeveloper.apple.com%2Fdocumentation%2Fxcode%2Fimproving_your_app_s_performance%2Freducing_your_app_s_launch_time%3Flanguage%3Dobjc)，他同时包含了Time Profile 以及 System Trace的功能，火焰图只抓了主线程（可以抓其他线程，但是查看没这么方便）并且还有一些非常隐晦的耗时操作也没法抓获，直接使用这个工具来做启动优化也是完全可行的,简单介绍以下这个工具的用法：

- 首先在Xcode的build settings 中Debug Information Format 设置为 DWARF with dsYM File (用于符号化地址)
- Xcode编译运行项目
- 通过 Xcode --> Open Developer Tool --> Instruments --> APP Launch 启动应用(这样可以直接运行debug包)，APPLaunch会启动应用5秒后自动关闭应用。
- 如果得到的分析数据没有符号化，在APP Launch选择屏幕左上角的file --> Symbols 选择亮绿灯的符号, 重新在在APP Launch运行项目。

![p4.webp](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/9f89865a51264aebba30352c4ea3bd55~tplv-k3u1fbpfcp-watermark.awebp)

大概是上图的操作方式，得出主线程的所有任务耗时时间，每个任务根据图右侧的堆栈挨个排查是否是启动链路中可优化的（几毫秒的也别放过）。

### 对找到的耗时任务进行修改

通过以上介绍的方案，可以找出可优化的任务，举几个可以借鉴优化的例子：

1. 懒加载/延后对应方法：在didFinishlanched方法较早的地方有挺多手动hook的方法，有一些是可以优化的，比如hook了路由的跳转，作用是启动之后在直播间相关组件没有初始化完成而执行进入直播间操作会导致异常，但是在启动时是没有路由操作的，这种hook可以延后到initialize方法第一次执行路由的时候。
2. 预加载图片：通过app luanch的动态图最后停留的部分，可以得到有21ms（而火焰图统计的在45ms左右）的耗时是在tabitem设置图片的时候,总共5个tab，10张图片。耗时主要是来自 `imageNamed:` 的解码操作。这个可以优化吗？

由于imageNamed方法是有缓存机制的，并且它也是线程安全的，所以可以在一个更早的时机将启动需要的图片在子线程进行解码。通过hook imageNamed方法得到启动时候所需的本地图片，在一个较早的时机进行 `图片预加载`：

```
// 目前我们是在appdelegate的didFinshedLaunch方法内执行
dispatch_async(dispatch_get_global_queue(0, 0), ^{
    NSArray *preloadImage = @[@"image1",@"image2"...];
    for (NSString *imageName in preloadImage) {
        [UIImage imageNamed:imageName];
    }
});
    
// 可以通过方案一的方式分别获取耗时来评估预加载是否有效，验证使用预加载之后耗时由40ms减少到了3ms
- (void)setAllTabbarItems
{
    CFAbsoluteTime start = CFAbsoluteTimeGetCurrent();
    [self setItemImage...];
    NSLog(@"setAllTabbarItems : %f",CFAbsoluteTimeGetCurrent() - start);
}
复制代码
```

**还有一个容易忽略的点**：我们的下拉刷新控件上面有一个图片动画组，进行解码也会很耗时，可以 `延后整个下拉刷新控件的设置` 到启动后而不是全部将图片丢到预加载。还有一些取数据库缓存、沙盒缓存的操作也可以提前到这个子线程预加载。

1. 延后自动登录：自动登录成功之后会发一个通知，有的地方收到这个通知之后会有拉配置等耗时的操作，在启动过程中是不需要自动登录的（如果首页的请求需要传uid之类的可以先缓存），把`自动登录逻辑放在启动完成之后`。因为我们的自动登录方式比较隐蔽且触发地方较多，在自动登录的位置打个断点，运行程序看启动流程中哪些步骤会导致登录操作，对其进行优化。
2. 预请求首页数据：通过火焰图分析，中间有两段较长时间主线程差不多处于空闲，是否可以优化？是因为主线程被其他线程挂起了吗？最终得出结论，是因为这时候在请求首页数据，等待数据渲染首页，首页的网络请求是在首页的viewdidload方法执行的，可以改到didFinishLaunchingWithOptions较早的时机`预请求首页数据`，缩短主线程空闲段的时间，在预请求的时候我们还要考虑一个网络资源竞争的问题，可以通过自定义的NSURLProtocol拦截找出启动时的所有NSURLSession请求，尽量保证首页的预请求为第一个请求，并且延后不必要的网络请求，我们有拦截到某sdk初始化时直接发了很多请求以及我们的IP直连相关逻辑，导致预请求首页的效果并不明显，（如何评估预请求效果？其实就是记录首页请求返回时时间点，然后减去main函数时间点得到从main-->数据返回的时间差）修改ip直连以及sdk的请求之后，首页数据返回提前了150-200ms，而在iPhone8plus上本身启动就1s多，启动速度直接就提升了15%。其次还有因为我们的首页使父子控制器的构造，在当前显示的子控制器加载的时候会去预加载/渲染左右两边的控制器，在启动流程中将这个步骤延后到启动完成（这个耗时也比较高）。
3. `缓存首页数据`:预请求可以提前数据返回时间，而使用缓存能直接去掉网络请求的耗时，常见的为先使用缓存再用请求的数据刷新界面，体验效果很差，如果直接就使用缓存则效果会很好，但是启动间隔太久会导致首页的主播大部分都已经下播了，于是我们给缓存设置了一个有效时期（目前定义为3-5分钟），如果本次启动距离上次缓存的数据时间相差不超过这个时期，则直接使用缓存，超过了则使用预加载的值。
4. `首页分段式加载`:我们的首页主要结构分为顶部的搜索框，以及下面的数据快，显然更重要的是下面数据快的展示，于是可以延后搜索框的加载，不过因为影响不太大（20ms）然后产品对这个方案不太支持，就没上了，如果你的app有这种明显的多个段落，也可以优先保证重要的段先展示出来。

### APP Launch 工具的威力

做完以上的优化之后，我们再使用APP Launch工具检测一下是否有其他可优化的地方，这里使用了system trace相关的功能。在检测的数据内点击下图的三角形，展开应用的所有线程，然后找到主线程。

![s4.webp](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/dd79eb92255e4c399b15ffe189534b96~tplv-k3u1fbpfcp-watermark.awebp) ![s3.webp](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/9fee38b4725b418b8e42c0b7df0c2a59~tplv-k3u1fbpfcp-watermark.awebp)

通过上图我们可以看到有一个等待锁的操作导致主线程被block了47ms（有时候测是80ms），我们需要找到原因然后处理它，比较简单的找的方式就是看看在主线程被block的这段时间，哪个子线程在执行任务，把工具检测到的线程都看一下，很容易就找到了某个子线程正在执行某个任务，而且存在中断->执行->中断这种反复调用中，我们对其进行修改，最终

![s2.webp](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/0eff5b01d2cb48949cb3fd1e4e9e69dc~tplv-k3u1fbpfcp-watermark.awebp) ![s1.webp](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ac875c3dc34d4dcf9c87312d3a851929~tplv-k3u1fbpfcp-watermark.awebp)

这段耗时`由188ms减少到了78ms`。其他的block也一并看了一下，系统行为无法调整。

## 5、点击push启动优化

将启动任务分为了高、中、低三个优先级，其中高优先级是应用启动必须的，中优先级定义为进直播间、跳转页面必须的，低优先级为启动完成后执行的任务。

优化点击push进落地页，其实也就是用前面介绍的方法优化中优先级的任务,其次通过push进直播间时，用户是期望优先看到直播内容，由于首页的请求、加载和渲染会占用资源，所以可以在push进直播间的链路上，将首页的请求延后至从直播间退出的时候。

## 6、二进制重排

二进制重排的原理：

![s5.webp](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d154d9293803467f99a7da29f89251c5~tplv-k3u1fbpfcp-watermark.awebp)

通过APP Launch检测缺页中断次数，由于应用启动后会在内存有缓存，所以需要重启设备清空内存缓存来检测。

![s6.webp](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/3e06d0efe8bc4f01bd8c113014674534~tplv-k3u1fbpfcp-watermark.awebp)

项目在编译生成二进制代码的时候，默认是按照链接的Object File(.o)顺序写文件，按照Object File内部的函数顺序写函数。 链接的顺序就是：build phases --> Compile Sources 里面的顺序。可以通过xcode设置 Build Settings --> Write Link Map File 为YES，生成Link map文件，然后在link map的# Symbols:段查看符号链接的顺序。

有了以上理论知识，我们实现二进制重排要做的就是在编译的时候，将启动需要的符号都排在一起，生成可执行文件，这样在分页加载到内存时尽量少的触发缺页中断。

- 如何调整项目编译时的符号顺序？XCode使用的链接器叫做ld，ld有个参数叫order_file，只要有这个文件并将文件的路径告诉XCode，XCode编译的时候就会按照文件中的符号顺序打包二进制可执行文件。
- 如何获取启动时需要的符号？其实就是获取启动时调用的所有方法，clang有提供对应的2个API[clang地址](https://link.juejin.cn?target=https%3A%2F%2Fclang.llvm.org%2Fdocs%2FSanitizerCoverage.html)，简单说就是在Other C falg 添加参数-fsanitize-coverage=func,trace-pc-guard,实现两个方法，__sanitizer_cov_trace_pc_guard_init，以及__sanitizer_cov_trace_pc_guard，第一个是初始化方法，第二个是每调用一个方法就会被拦截到，然后记录下启动时拦截到的所有方法，这样就获取到了启动时所需要的符号，将符号写入并生成order_file文件，在Build Settings -->Order file将文件路径设置进去。

之后可以通过分析linkmap的# Symbols:段确认符号是否有调整，确认有调整之后对成果进行检验：在iOS13 iPhone8plus上无论是检测的page fault次数/耗时，还是启动耗时，使用二进制重排与不使用相差很小很小，大概就是每次测量的波动范围内，不知道是否是iOS13 的dyld2升级到dyld3已经优化过了（有关dyld升级优化感兴趣可以自行搜索了解）。所以最终我们也是放弃了二进制重排。

## 7、启动优化后续计划

1. 启动模块化，目前所有的启动项都集中在一个类里面，光+import头文件就200行，所以在下个版本会将启动项按业务分成多个模块进行处理。
2. 推动其他sdk进行优化，特别是子线程占用较多的需要控制一下线程数量，目前相关sdk也在处理中。

# 四、启动监控

## 流程

为了可以监控到日常开发过程中启动耗时变化，监控了启动过程中的方法调用耗时，通过每天构建对比当天版本和昨天版本的差异分析耗时原因，流程如下：

![s7.webp](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b1fe648bf16f433186094718c4309b34~tplv-k3u1fbpfcp-watermark.awebp)

- Jenkins 编译构建，构建完成后，上报 LinkMap
- 打包完成后，通过 `ios-deploy`，真机安装 App
- 启动 `Appium`, 用于多次启动 App
- 运行测试脚本，通过控制 Appium, Appium 控制设备，重复冷启动多次，上报数据，取平均值，减少浮动影响
- 分析数据，耗时新增，减少，增加和 Diff 等
- 分析结果邮件发送
- 优化代码

## 分析报告

第一部分是 `Pre-Main` 和 首页图片加载完成耗时, 如下: ![img](https://user-gold-cdn.xitu.io/2020/6/19/172ca58f44f0a1ce?imageView2/2/w/1304/q/85/format/webp/interlace/1)

![s8.webp](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b161d2de6fbf4c7aab10612ef6875212~tplv-k3u1fbpfcp-watermark.awebp)

第二部分是通过对比两个版本的启动耗时数据进行 Diff, 启动过程中，如果当前版本的方法在对比版本没有出现，就认为是新增方法

![g1.webp](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/bbb4dfa7b5ed4317bfde37dbbf43b136~tplv-k3u1fbpfcp-watermark.awebp)

第三部分是已存在方法耗时变化

![g2.webp](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e2f252e6ae854b45bb5a040619a27f75~tplv-k3u1fbpfcp-watermark.awebp)

第四部分是库在启动过程中，占用的耗时

![g3.webp](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e94e77e41b864afc86ef33119ea1c402~tplv-k3u1fbpfcp-watermark.awebp)

第五部分是 + load 方法，占用的耗时

![g4.webp](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/0481a1369b0e4445a31ef80046003a67~tplv-k3u1fbpfcp-watermark.awebp)

## 实现

通过 Hook 记录启动阶段方法和对应方法的耗时

### 统计 Pre-Main 和首页图片加载完成耗时

Pre-Main 耗时 = 进入 main 函数的时间 - 进程创建时间，以下是获取进程创建时间实现

```
+ (BOOL)processInfoForPID:(int)pid procInfo:(struct kinfo_proc *)procInfo {
    int cmd[4] = {CTL_KERN, KERN_PROC, KERN_PROC_PID, pid};
    size_t size = sizeof(*procInfo);
    return sysctl(cmd, sizeof(cmd)/sizeof(*cmd), procInfo, &size, NULL, 0) == 0;
}

+ (NSTimeInterval)processStartTime {
    struct kinfo_proc kProcInfo;
    if ([self processInfoForPID:[[NSProcessInfo processInfo] processIdentifier] procInfo:&kProcInfo]) {
        return kProcInfo.kp_proc.p_un.__p_starttime.tv_sec * 1000.0 + kProcInfo.kp_proc.p_un.__p_starttime.tv_usec / 1000.0;
    } else {
        NSAssert(NO, @"无法取得进程的信息");
        return 0;
    }
}
复制代码
```

首页图片加载完成耗时：Hook 图片下载方法，在启动完成前将所有调用该方法的 URL 存入数组，图片下载完成之后移除数组，当数组内元素个数为 0 时，代表首页第一屏的图片下载完成，即为结束时间

### Pre-Main 阶段的 + load 方法、C++ static constructors 、 **attribute**((constructor))、 __mod_init_func section 中的函数和 OC 方法耗时统计

#### + load

项目中的 `+ load` 方法或多或少对启动耗时有一定的影响，通过 `Hook + load` 方法，统计 `+ load` 方法耗时, 主要是通过一个比 + load 方法执行还要早的时机，对定义了 load 方法的类进行 Hook, 对 `load` 方法的前后插入统计耗时的处理

`mach-o`  中 `__DATA,__objc_nlclslist` 和 `__DATA,__objc_nlcatlist` 这两个 `Section` 分别保存了 `non lazy class` 和 `non lazy cateogry`, 定义 load 方法的类和分类, 通过 `getsectbynamefromheader` 把定义了 load 方法的类和分类获取出来进行 Hook, 用最早加载的动态库里定义类的 `load` 方法，比主二进制的 `load` 方法调用还要早。通过在动态库中 `load` 方法这个时机进行 Hook

![g5.webp](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/7bf6458c7d27436baf51c8474df52bf1~tplv-k3u1fbpfcp-watermark.awebp)

```
struct Category {
    char * _Nonnull category_name;
    char * _Nonnull class_name;
    struct objc_method_list * _Nullable instance_methods;
    struct objc_method_list * _Nullable class_methods;
    struct objc_protocol_list * _Nullable protocols;
}  

// 获取 load 方法的类和分类
const section *nonLazyClass = GetSectByNameFromHeader((void *)mach_header, "__DATA", "__objc_nlclslist");
if (NULL != nonLazyClass) {
    for (ptr address = nonLazyClass->offset; address < nonLazyClass->offset + nonLazyClass->size; address += sizeof(const void *)) {
        Class cls = (__bridge Class)(*(void **)(mach_header + address));
    }
}
    
const section *nonLazyCategory = GetSectByNameFromHeader((void *)mach_header, "__DATA", "__objc_nlcatlist");
if (NULL != nonLazyCategory) {
    for (ptr address = nonLazyCategory->offset; address < nonLazyCategory->offset + nonLazyCategory->size; address += sizeof(const void **)) {
        struct Category *cat = (*(struct Category **)(mach_header + address));
    }
}

// 遍历 load class 和对应 category 的 MethodList 进行 Hook
IMP originIMP = loadMethod->imp;
IMP replaceIMP = imp_implementationWithBlock(^(__unsafe_unretained id self, SEL sel) {
    ((void (*)(id, SEL))originIMP)(self, sel);
});
loadMethod->imp = replaceIMP;
复制代码
```

#### objc_msgSend

OC 的方法执行过程会调用到 `objc_msgSend`, 所以对其进行 Hook，能统计到 OC 方法的耗时，`objc_msgSend` 是变参函数，通过保存现场，保持参数不变，调用原来的 objc_msgSend,  参考 [InspectiveC](https://link.juejin.cn?target=https%3A%2F%2Fgithub.com%2FDavidGoldman%2FInspectiveC%2Fblob%2Fmaster%2FInspectiveCarm64.mm) 实现

```
static void replacementObjc_msgSend() {
  __asm__ volatile (
      // 保存 q0-q7 
      "stp q6, q7, [sp, #-32]!\n"
      "stp q4, q5, [sp, #-32]!\n"
      "stp q2, q3, [sp, #-32]!\n"
      "stp q0, q1, [sp, #-32]!\n"
      // 保存 x0-x8, lr
      "stp x8, lr, [sp, #-16]!\n"
      "stp x6, x7, [sp, #-16]!\n"
      "stp x4, x5, [sp, #-16]!\n"
      "stp x2, x3, [sp, #-16]!\n"
      "stp x0, x1, [sp, #-16]!\n"
      "mov x2, x1\n"
      "mov x1, lr\n"
      "mov x3, sp\n"
      // 调用 preObjc_msgSend
      "bl __Z15preObjc_msgSendP11objc_objectmP13objc_selectorP9RegState_\n"
      "mov x9, x0\n"
      "mov x10, x1\n"
      "tst x10, x10\n"
      // 读取 x0-x8, lr
      "ldp x0, x1, [sp], #16\n"
      "ldp x2, x3, [sp], #16\n"
      "ldp x4, x5, [sp], #16\n"
      "ldp x6, x7, [sp], #16\n"
      "ldp x8, lr, [sp], #16\n"
      // 读取 q0-q7
      "ldp q0, q1, [sp], #32\n"
      "ldp q2, q3, [sp], #32\n"
      "ldp q4, q5, [sp], #32\n"
      "ldp q6, q7, [sp], #32\n"
      "b.eq Lpassthrough\n"
      // blr 调用原始 objc_msgSend
      "blr x9\n"
      // 保存 x0-x9
      "stp x0, x1, [sp, #-16]!\n"
      "stp x2, x3, [sp, #-16]!\n"
      "stp x4, x5, [sp, #-16]!\n"
      "stp x6, x7, [sp, #-16]!\n"
      "stp x8, x9, [sp, #-16]!\n"
      // 保存 q0-q7
      "stp q0, q1, [sp, #-32]!\n"
      "stp q2, q3, [sp, #-32]!\n"
      "stp q4, q5, [sp, #-32]!\n"
      "stp q6, q7, [sp, #-32]!\n"
      // 调用 postObjc_msgSend hook.
      "bl __Z16postObjc_msgSendv\n"
      "mov lr, x0\n"
      // 读取 q0-q7
      "ldp q6, q7, [sp], #32\n"
      "ldp q4, q5, [sp], #32\n"
      "ldp q2, q3, [sp], #32\n"
      "ldp q0, q1, [sp], #32\n"
       // 读取 x0-x9
      "ldp x8, x9, [sp], #16\n"
      "ldp x6, x7, [sp], #16\n"
      "ldp x4, x5, [sp], #16\n"
      "ldp x2, x3, [sp], #16\n"
      "ldp x0, x1, [sp], #16\n"
      "ret\n"
      "Lpassthrough:\n"
      "br x9"
    );
}
复制代码
```

#### C++ static constructors 、 attribute((constructor))、 _modinit_func section 中的函数

`__mod_init_func` 存储初始化相关的函数地址, `__mod_init_func` 是在 DATA 段，Pointer 指向的区域是 TEXT 段, 项目中的这类函数很多，这些函数会在 Pre-Main 阶段执行，但是基本都不耗时, 通过 getsectiondata(machHeader, "__DATA", "__mod_init_func", &size)，读取函数指针，用 hook 函数指针替换原来的函数指针，把原来的函数地址记录在全局数组中，hook 函数从数组中根据 index 调用本该执行的函数

![g6.webp](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ed100f1e323443e7818275df5d765a9a~tplv-k3u1fbpfcp-watermark.awebp)

```
void myinit(int argc, char **argv, char **envp) {}

__attribute__((section("__DATA, __mod_init_func"))) typeof(myinit) *__init = myinit;

TestClass test = TestClass();

__attribute__((constructor)) void testConstructor() {}
复制代码
void HookInitFuncInitializer(int argc, const char *argv[], const char *envp[], const char *apple[], const struct ProgramVarsStr *vars) {
    ++CurrentPointerIndex;
    InitializerType f = (InitializerType)Initializer[CurrentPointerIndex];
    f(argc, argv, envp, apple, vars);
    
    NSString *symbol = [NSString stringWithFormat:@"%p", f];
    Dl_info info;
    if (0 != dladdr(f, &info)) {
        NSString *sname = @(info.dli_sname);
        if (sname.length > 0) {
            symbol = sname;
        }
    }
}

static void HookModInitFunc() {
    Dl_info info;
    dladdr(HookModInitFunc, &info);
    mach_header *machHeader = info.dli_fbase;
    unsigned long size = 0;
    pointer *p = (pointer *)getsectiondata(machHeader, "__DATA", "__mod_init_func", &size);
    int count = (int)(size / sizeof(void *));
    for (int i = 0; i < count; ++i) {
        pointer ptr = p[i];
        Initializer[i] = ptr;
        p[i] = (pointer)HookInitFuncInitializer;
    }
}
复制代码
```

#### 库耗时统计

LinkMap 中取到 Object files 部分，获取到 `libAFNetworking.a(AFHTTPSessionManager.o)` 部分，然后解析成 `AFNetworking` 和 `AFHTTPSessionManager`，通过这种方式能粗略的统计到是那个库，库里有包含的类，进而统计出那个方法属于该库，这个方法统计不到类的命名不对应文件名，或常见 Category 那些情况等

```
.../Products/Debug-iphoneos/AFNetworking/libAFNetworking.a(AFHTTPSessionManager.o)
复制代码
```



