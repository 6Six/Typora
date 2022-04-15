# Flutter编译原理



### Flutter简介

![img](http://techforum-img.cn-hangzhou.oss-pub.aliyun-inc.com/1530849034199/1.jpg)



Flutter的架构主要分为三层：Framework，Engine和Embedder。

- Framework
  - 使用dart实现，包括Material Design风格的Widget，Cupertino（针对iOS）风格的Widgets，文本/图片/按钮等基础Widgets、渲染、动画、手势等。
  - **核心代码：** **flutter**仓库下的**flutter package**，以及**sky_engine**仓库下的**io**、**async**、**ui**（dart：ui 库提供了Flutter框架和引擎之间的接口）等package；

- Engine
  - 使用C++实现，主要包括：Skia、Dart和Text
    - Skia是开源二维图形库，提供了适用于多种软硬件平台的通用API。已经用于Google Chrome、ChromeOS、Android、Mozilla Firefox、FirefoxOS等产品的图形引擎，支持平台包括Windows7+、MacOS 10.10.5+、iOS8+、Android4.1+、Ubuntu14.04+等；
    - Dart部分主要包括：Dart runtime、Garbage Collection（GC），如果是Debug模式的话，还包括JIT（Just In Time）支持。Release和Profile模式下，是AOT（Ahead Of Time）编译成了原生的arm代码，并不存在JIT部分。Text即文本渲染，其渲染层次如下：衍生自minikin的libtxt（用于字体选择、分隔行）。HartBuzz用于字形选择和成型。Skia作为渲染/GPU后端，在Android和Fuchsia上使用FreeType渲染，在iOS上使用CoreGraphics来渲染字体；
    - Embedder是一个嵌入层，即把Flutter嵌入到各个平台上去，这里做的主要工作包括渲染Surface设置、线程设置以及插件等。从这里可以看出，Flutter的平台相关层次很低，平台（如iOS）只是提供一个画布，剩余的所有渲染相关的逻辑都在Flutter内部，这就使得它具有了很好的跨端一致性。



[Tips：AOT和JIT及混合编译的区别、优劣](https://github.com/codeegginterviewgroup/CodeEggDailyInterview/blob/master/Android%20%E8%BF%9B%E9%98%B6/76.AOT%E5%92%8CJIT%E4%BB%A5%E5%8F%8A%E6%B7%B7%E5%90%88%E7%BC%96%E8%AF%91%E7%9A%84%E5%8C%BA%E5%88%AB%E3%80%81%E4%BC%98%E5%8A%A3.md)



***



### Flutter模式

Flutter支持常用的debug、release、profile等模式，但它又有不一样的地方。

- Debug模式
  - 对应Dart的JIT模式，又称检查模式或慢速模式；
  - 支持设备，模拟器（iOS/Android），此模式下打开了断言，包括所有的调试信息，服务扩展和Observatory等调试辅助；
  - 此模式为快速开发和运行做了优化，但并未对执行速度、包大小和部署做优化；
  - Debug模式下编译使用JIT技术，支持hot reload；
- Release模式
  - 对应了Dart的AOT模式，此模式目标即为部署到终端用户；
  - 只支持真机，不包括模拟器；
  - 关闭了所有断言，尽可能多的去掉了调试信息，关闭了所有调试工具；
  - 为快速启动，快速执行，包大小做了优化；
  - 禁止了所有调试辅助手段，服务扩展；
- Profile模式
  - 类似release模式，只是多了对于Profile模式的服务扩展的支持，支持跟踪，以及最小化使用跟踪信息需要的依赖，例如：observatory可以连接上进程；
  - Profile不支持模拟器的原因：模拟器上的诊断并不代表真实的性能；



事实上flutter下的iOS/Android工程本质上依然是一个标准的iOS/Android工程，flutter只是通过在BuildPhase中添加shell来生成和嵌入App.framework和Flutter.framework（iOS），通过graddle来添加flutter.jar和vm/isolate_snapshot_data/instr(Android)来讲flutter相关代码编译和嵌入原生App而已。

因此以下主要讨论因flutter引入的构建、运行等原理，编译target为arm相关。



---



### Flutter代码的编译与运行（iOS）



##### Release模式下的编译

---

![img](http://techforum-img.cn-hangzhou.oss-pub.aliyun-inc.com/1530849053996/3.jpg)



其中gen_snapshot是dart编译器，采用了tree shaking（类似依赖树逻辑，可生成最小包，也因而在Flutter中禁止了dart支持的反射特性）等技术，用于生成汇编形式的机器代码，再通过xcrun等编译工具链生成最终的App.framework。

换句话说，所有的dart代码，包括业务代码、三方package代码，它们所依赖的flutter框架代码，最终将会变成App.framework。



tree shaking功能位于gen_snapshot中，对应逻辑参见：engine/src/third_party/dart/runtime/vm/compiler/aot/precompiler.cc

dart代码最终对应到App.framework中的符号如下所示：

![img](http://techforum-img.cn-hangzhou.oss-pub.aliyun-inc.com/1530849063503/4.jpg)



事实上，类似Android Release下的产物（见下文），App.framework也包含了kDartVmSnapshotData，KDartVMSnapshotInstructions，KDartIsolateSnapshotData，KDartIsolateSnapshotInstructions四个部分。为什么iOS使用App.framework这种方式，而不是Android的四个文件的方式呢？原因在于iOS下，因为系统的限制，Flutter引擎不能够在运行时将某内存页标记为可执行，而Android是可以的。



Flutter.framework对应了Flutter架构中的engine部分，以及Embedder。实际中Flutter.framework位于flutter仓库的 /bin/cache/artifacts/engine/ios*下，默认从google仓库拉取。当需要自定义修改的时候，可通过下载engine源码，利用Ninja构建系统来生成。



Flutter相关代码的最终产物是： App.framework（dart代码生成）和Flutter.framework（引擎）。从Xcode工程的视角看，Generated.xcconfig描述了Flutter相关环境的配置信息，然后Runner工程设置中的Build Phase新增的xcode_backend.sh实现了Flutter.framework的拷贝（从Flutter仓库的引擎到Runner工程根目录下的Flutter目录）与嵌入和App.framework的编译与嵌入。最终生成的Runner.app中Flutter相关内容如下所示：

![img](http://techforum-img.cn-hangzhou.oss-pub.aliyun-inc.com/1530849073437/5.jpg)



其中flutter_assets是相关的资源，代码则是位于Frameworks下的App.framework和Flutter.framework。

---



##### Release模式下的运行

---

Flutter相关的渲染，事件，通信处理逻辑如下所示：

![img](http://techforum-img.cn-hangzhou.oss-pub.aliyun-inc.com/1530849085037/6.jpg)



其中dart中的main函数调用栈如下：

![img](http://techforum-img.cn-hangzhou.oss-pub.aliyun-inc.com/1530849093619/7.jpg)

---



##### Debug模式下的编译

Debug模式下flutter的编译，结构类似Release模式，差异主要表现为两点：

- Flutter.framework
  - 因为是Debug，此模式下Framework中是有JIT支持的，而在Release模式下并没有JIT部分；
- App.framework
  - 不同于AOT模式下的App.framework是Dart代码对应的本地机器代码，JIT模式下，App.framework只有几个简单的API，其Dart代码存在于snapshot_blob.bin文件里，这部分的snapshot是脚本快照，里面是简单的标记化的源代码；
  - 所有的注释，空白字符都被移除，常量也被规范化，也没有机器码，tree shaking或者是混淆；
  - App.framework中的符号表如下所示：
    ![img](http://techforum-img.cn-hangzhou.oss-pub.aliyun-inc.com/1530849107216/8.jpg)
  - 对Runner.app/flutter_assets/snapshot_blob.bin执行strings命令可以看到如下内容：
    ![img](http://techforum-img.cn-hangzhou.oss-pub.aliyun-inc.com/1530849133399/9.jpg)
  - Debug模式下main入口的调用堆栈如下：
    ![img](http://techforum-img.cn-hangzhou.oss-pub.aliyun-inc.com/1530849145312/10.jpg)



---



###   Flutter代码的编译与运行（Android） 

鉴于Android和iOS除了部分平台相关的特性外，其他逻辑如Release对应AOT，Debug对应JIT等均类似，此处指涉及两者不同。



##### Release模式下的编译

---

release模式下，flutter下Android工程中dart代码整个构建链路如下所示：

![img](http://techforum-img.cn-hangzhou.oss-pub.aliyun-inc.com/1530849153957/11.jpg)



其中vm/isolate_snapshot_data/instr内容均为arm指令，将会在运行时被engine载入，并标记 vm/isolate_snapshot_instr 为可执行。vm_中涉及runtime等服务（如gc），用于初始化 DartVM，调用入口见 Dart_initialize(dart.api.h)。isolate__ 则是对应了我们App的代码，用于创建一个新isolate，调用入口见 Dart_CreateIsolate(dart_api.h)。



flutter.jar类似iOS的Flutter.framework，包括了engine部分的代码（Flutter.jar中的libFlutter.so），以及一套将Flutter嵌入Android的类和接口（FlutterMain，FlutterView，FlutterNativeView等）。实际中flutter.jar位于flutter仓库的 /bin/cache/artifacts/engine/android* 下，默认从google仓库拉取。当需要自定义修改的时候，可通过下载engine源码，利用Ninja构建系统来生成flutter.jar。



以isolate_snapshot_data/instr为例，执行disarm命令结果如下：

![img](http://techforum-img.cn-hangzhou.oss-pub.aliyun-inc.com/1530849163484/12.jpg)

![img](http://techforum-img.cn-hangzhou.oss-pub.aliyun-inc.com/1530849172376/13.jpg)



其Apk结构如下所示：

![img](http://techforum-img.cn-hangzhou.oss-pub.aliyun-inc.com/1530849183954/14.jpg)



Apk新安装之后，会根据一个ts的判断（packageinfo中的versionCode结合lastUpdateTime）来决定是否拷贝Apk中的assets，拷贝后内容如下所示：

![img](http://techforum-img.cn-hangzhou.oss-pub.aliyun-inc.com/1530849202204/15.jpg)



isolate/vm_snapshot_data/instr 均最后位于App的本地data目录下，而这部分又属于可写内容，因此可以通过下载并替换的方式，完成App的整个替换和更新。



----



##### Release模式下的运行

---

![img](http://techforum-img.cn-hangzhou.oss-pub.aliyun-inc.com/1530849214403/16.jpg)



---

Debug模式下的编译

---

类似iOS的Debug/Release的差别，Android的Debug和Release的差异主要包括以下两部分：

- flutter.jar：  区别同iOS
- App代码部分：位于flutter_assets下的snapshot_blob.bin，同iOS。



在介绍了iOS/Android下的Flutter编译原理后，下面着重描述下如何定制flutter/engine以完成定制和优化。鉴于Flutter处于敏捷的迭代中，现在的问题后续不一定是问题，因而此部分并不是要去解决多少问题，而是选取不同类别的问题来说明解决思路。



---

### Flutter构建相关的定制和优化

---

Flutter是一个很复杂的系统，除了上述提到的三层架构中的内容外，还包括Flutter Android Studio(Intellij)插件，pub仓库管理等。但我们的定制和优化往往是在flutter的工具链相关，具体代码位于flutter仓库的flutter_tools包。接下来举例说明下如何对这部分做定制。



---



##### Android部分

---

相关内容包括flutter.jar，libflutter.so(位于flutter.jar下)，gen_snapshot，flutter.gradle，flutter(flutter_tools)。



1. 限定Android中target为armeabi

   此部分属于构建相关，逻辑位于flutter.gradle下。当App是通过armeabi支持armv7/arm64的时候，需要修改flutter的默认逻辑。如下所示：
   ![img](http://techforum-img.cn-hangzhou.oss-pub.aliyun-inc.com/1530849223750/17.jpg)

   因为gradle本身的特点，此部分修改后直接构建即可生效。

2. 设定Android启动时默认使用第一个launchable-activity，此部分属于flutter_tools相关，修改如下：
   ![img](http://techforum-img.cn-hangzhou.oss-pub.aliyun-inc.com/1530849232169/18.jpg)
   这里的重点不是如何去修改，而是如何去让修改生效。原理上来说，flutter run/build/analyze/test/upgrade等命令实际上执行的都是flutter(flutter_repo_dir/bin/flutter)这一脚本，再通过脚本通过dart执行flutter_tools.snapshot(通过packages/flutter_tools生成)。其逻辑如下：

   ![img](http://techforum-img.cn-hangzhou.oss-pub.aliyun-inc.com/1530849241293/19.jpg)

   不难看出要重新构建flutter_tools，可以删除flutter_repo_dir/bin/cache/flutter_tools.stamp(这样重新生成一次)，或者屏蔽掉if/fi判断(每一次都会重新生成)。

3. 如何在Android工程Debug模式下使用release模式的flutter

   当开发者在研发中发现flutter有些卡顿时，猜测可能是逻辑的原因，也可能是因为是Debug下的flutter。此时可以构建release下的apk，也可以将flutter强制修改为release模式如下：
   ![img](http://techforum-img.cn-hangzhou.oss-pub.aliyun-inc.com/1530849249147/20.jpg)



---

##### iOS部分

---

相关内容包括:Flutter.framework，gen_snapshot，xcode_backend.sh，flutter(flutter_tools)。



1. 优化构建过程中反复替换Flutter.framework导致的重新编译

   此部分逻辑属于构建相关，位于xcode_backend.sh中，Flutter为了保证每次获取到正确的Flutter.framework,每次都会基于配置(见Generated.xcconfig配置)查找和替换Flutter.framework，但这也导致了工程中对此Framework有依赖部分代码的重新编译，修改如下：
   ![img](http://techforum-img.cn-hangzhou.oss-pub.aliyun-inc.com/1530849258232/21.jpg)

2. 如何在iOS工程Debug模式下使用release模式的flutter

   只需要将Generated.xcconfig中的FLUTTER_BUILD_MODE修改为release，FLUTTER_FRAMEWORK_DIR修改为release对应的路径即可。

3. armv7的支持

   [原始文章](https://github.com/flutter/engine/wiki/iOS-Builds-Supporting-ARMv7)

   事实上flutter本身是支持iOS下的armv7的，但目前并未提供官方支持，需要自行修改相关逻辑，具体如下：

   - 默认的逻辑可以生成
     Flutter.framework(arm64)

   - 修改flutter以使得flutter_tools可以每次重新构建，修改build_aot.dart和mac.dart，将相关针对iOS的arm64修改为armv7,修改gen_snapshot为i386架构。

     其中i386架构下的gen_snapshot可通过以下命令生成：
     ![img](http://techforum-img.cn-hangzhou.oss-pub.aliyun-inc.com/1530849267730/22.jpg)

     这里有一个隐含逻辑：

     构建gen_snapshot的CPU相关预定义宏(x86_64/__i386等)，目标gen_snapshot的arch，最终的App.framework的架构整体上要保持一致。即x86_64->x86_64->arm64或者i386->i386->armv7。

   - 在iPhone4S上，会发生因gen_snapshot生成不被支持的SDIV指令而造成EXC_BAD_INSTRUCTION(EXC_ARM_UNDEFINED)错误，可通过给gen_snapshot添加参数--no-use-integer-division实现(位于build_aot.dart)。其背后的逻辑如下图所示：

     ![img](http://techforum-img.cn-hangzhou.oss-pub.aliyun-inc.com/1530849278631/23.jpg)

   - 基于a和b生成的Flutter.framework,将其lipo create生成同时支持armv7和arm64的Flutter.framework。

   - 修改Flutter.framework下的Info.plist，移除

     ![img](http://techforum-img.cn-hangzhou.oss-pub.aliyun-inc.com/1530849295750/24.jpg)

     同理，对于App.framework也要作此操作，以免上架后会受到App Thining的影响。



---

##### flutter_tools的调试

---

例如我们想了解flutter在构建debug模式下的apk的时候，具体执行的逻辑如何，可以按照下面的思路走：

- 了解flutter_tools的命令行参数
  ![img](http://techforum-img.cn-hangzhou.oss-pub.aliyun-inc.com/1530849304722/25.jpg)
- 以dart工程形式打开packages/flutter_tools，基于获得的参数修改flutter_tools.dart，设置命令行dart app即可开始调试。
  ![img](http://techforum-img.cn-hangzhou.oss-pub.aliyun-inc.com/1530849313756/26.jpg)



---

##### 定制engine与调试

----

假设我们在flutter beta v0.3.1的基础上进行定制与业务开发，为了保证稳定，一定周期内并不升级SDK，而此时，flutter在master上修改了某个v0.3.1上就有的bug，记为fix_bug_commit。如何才能跟踪和管理这种情形呢？

- flutter beta v0.3.1指定了其对应的engine commit为:09d05a389，见 flutter/bin/internal/engine.version。

- 获取engine代码

- 因为2中拿到的是master代码，而我们需要的是特定commit(09d05a389)对应的代码库，因而从此commit拉出新分支:custom_beta_v0.3.1。

- 基于custom_beta_v0.3.1(commit:09d05a389)，执行gclient sync，即可拿到对应flutter beta v0.3.1的所有engine代码。

- 使用git cherry-pick fix_bug_commit将master的修改同步到custom_beta_v0.3.1，如果修改有很多对最新修改的依赖，可能会导致编译失败。

- 对于iOS相关的修改执行以下代码：
  ![img](http://techforum-img.cn-hangzhou.oss-pub.aliyun-inc.com/1530849324084/27.jpg)

  如果需要调试Flutter.framework源代码，构建的时候命令如下：

  ![img](http://techforum-img.cn-hangzhou.oss-pub.aliyun-inc.com/1530849331432/28.jpg)

  用生成产物替换掉flutter中的Flutter.framework和gen_snapshot，即可调试engine源代码。

- 对于Android相关的修改执行以下代码：
  ![img](http://techforum-img.cn-hangzhou.oss-pub.aliyun-inc.com/1530849340608/29.jpg)

  即可生成针对Android的arm&debug/release/profile 的产物。可用构建产物替换

  flutter/bin/cache/artifacts/engine/android*下的gen_snapshot和flutter.jar。

----



#### 参考文档：

1. [Flutter's modes](https://github.com/flutter/flutter/wiki/Flutter%27s-modes)
2. [iOS Builds Supporting ARMv7](https://github.com/flutter/engine/wiki/iOS-Builds-Supporting-ARMv7)
3. [Contributing to the Flutter engine](https://github.com/flutter/engine/blob/master/CONTRIBUTING.md)
4. [Flutter System Architecture](https://docs.google.com/presentation/d/1cw7A4HbvM_Abv320rVgPVGiUP2msVs7tfGbkgdrTy0I/edit)
5. [The magic of flutter](https://docs.google.com/presentation/d/1B3p0kP6NV_XMOimRV09Ms75ymIjU5gr6GGIX74Om_DE/edit)
6. [Symbolicating production crash stacks](https://github.com/flutter/engine/wiki/Symbolicating-production-crash-stacks)
7. [flutter.io](https://flutter.io/docs)
8. [获取本文使用的源代码](https://github.com/FlutterRepo/hello_flutter.git)

























































