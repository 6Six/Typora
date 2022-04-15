# [Flutter包大小治理上的探索与实践](https://tech.meituan.com/2020/09/18/flutter-in-meituan.html)	



## 一、背景

Flutter作为一种全新的响应式、跨平台、高性能的移动开发框架，在性能、稳定性和多端体验一致上都有着较好的表现，自开源以来，已经受到越来越多开发者的喜爱。随着Flutter框架的不断发展和完善，业内越来越多的团队开始尝试并落地Flutter技术。不过在实践过程中我们发现，Flutter的接入会给现有的应用带来比较明显的包体积增加。不论是在Android还是在iOS平台上，仅仅是接入一个Flutter Demo页面，包体积至少要增加5M，这对于那些包大小敏感的应用来说其实是很难接受的。

对于包大小问题，Flutter官方也在持续跟进优化：

- Flutter V1.2 开始支持[Android App Bundles](https://developers.googleblog.com/2019/02/launching-flutter-12-at-mobile-world.html)，支持Dynamic Module下发。
- Flutter V1.12 [优化了2.6%](https://medium.com/flutter/announcing-flutter-1-12-what-a-year-22c256ba525d) Android平台Hello World App大小（3.8M -> 3.7M）。
- Flutter V1.17 通过优化[Dart PC Offset存储](https://github.com/dart-lang/sdk/commit/a2bb7301c5795e6b28089a8dc96e6ab5ca798e22)以减少StackMap大小等多个手段，再次优化了产物大小，实现[18.5%的缩减](https://medium.com/flutter/announcing-flutter-1-17-4182d8af7f8e)。
- Flutter V1.20 通过[Icon font tree shaking](https://github.com/flutter/flutter/pull/49737)移除未用到的icon fonts，进一步优化了应用大小。

除了Flutter SDK内部或Dart实现的优化，我们是否还有进一步优化的空间呢？答案是肯定的。为了帮助业务方更好的接入和落地Flutter技术，MTFlutter团队对Flutter的包大小问题进行了调研和实践，设计并实现了一套基于动态下发的包大小优化方案，瘦身效果也非常可观。这里分享给大家，希望对大家能有所帮助或者启发。

## 二、Flutter包大小问题分析

在Flutter官方的优化文档中，提到了减少应用尺寸的方法：在V1.16.2及以上使用—split-debug-info选项（可以分离出debug info）；移除无用资源，减少从库中带入的资源，控制适配的屏幕尺寸，压缩图片文件。这些措施比较直接并容易理解，但为了探索进一步瘦身空间并让大家更好的理解技术方案，我们先从了解Flutter的产物构成开始，然后再一步步分析有哪些可行的方案。

### 2.1 Flutter产物介绍

我们首先以官方的Demo为例，介绍一下Flutter的产物构成及各部分占比。不同Flutter版本以及打包模式下，产物有所不同，本文均以Flutter 1.9 Release模式下的产物为准。

**2.1.1 iOS侧Flutter产物**

![图1 Flutter iOS 产物组成示意图](https://p0.meituan.net/travelcube/595ef0b84bd765cad2f6ff01c315b69949858.png@690w_263h_80q)

图1 Flutter iOS 产物组成示意图



iOS侧的Flutter产物主要由四部分组成（info.plist 比较小，对包体积的影响可忽略，这里不作为重点介绍），表格1中列出了各部分的详细信息。

![表1 Flutter产物组成](https://p0.meituan.net/travelcube/a9638d07729fbc3c6334a3ae3d4c261b96573.png@1152w_536h_80q)

表1 Flutter产物组成



2.1.2 Android侧Flutter产物

![图2 Flutter Android 产物组成示意图](https://p1.meituan.net/travelcube/6b4a69a925c331442735fa593c86dbc775094.png@870w_362h_80q)

图2 Flutter Android 产物组成示意图



Android侧的Flutter产物总共5.16MB，由四部分组成，表格2中列出了各部分的详细信息。

![表2 Flutter Android产物组成](https://p0.meituan.net/travelcube/c65596afd6e649bd204e53ae5761451268881.png@1316w_414h_80q)

表2 Flutter Android产物组成



**2.1.3 各部分产物的变化趋势**

无论是Android还是iOS，Flutter的产物大体可以分为三部分：

1. Flutter引擎，该部分大小固定不变，但初始占比较高。
2. Flutter业务与框架，该部分大小随着Flutter业务代码的增多而逐渐增加。它是这样的一个曲线：初始增长速度极快，随着代码增多，增长速度逐渐减缓，最终趋近线性增长。原因是Flutter有一个Tree Shaking机制，从Main方法开始，逐级引用，最终没有被引用的代码，比如类和函数都会被裁剪掉。一开始引入Flutter之后随便写一个业务，就会大量用到Flutter/Dart SDK代码，这样初期Flutter包体积极速增加，但是过了一个临界点，用户包体积的增加就基本取决于Flutter业务代码增量，不会增长得太快。
3. Flutter资源，该部分初始占比较小，后期增长主要取决于用到的本地图片资源的多少，增长趋势与资源多少成正比。

下图3展示了Flutter各资源变化的趋势：

![图3 Flutter各资源大小变化的趋势图](https://p0.meituan.net/travelcube/0c6b01a4295b35363ff9f41aaeed4f3c113875.png@1118w_820h_80q)

图3 Flutter各资源大小变化的趋势图



### 2.2 不同优化思路分析

上面我们对Flutter产物进行了分析，接下来看一下官方提供的优化思路如何应用于Flutter产物，以及对应的困难与收益如何。

**1.删减法**

Flutter引擎中包括了Dart、skia、boringssl、icu、libpng等多个模块，其中Dart和skia是必须的，其他模块如果用不到倒是可以考虑裁掉，能够带来几百k的瘦身收益。业务方可以根据业务诉求自定义裁剪。

Flutter业务产物，因为Flutter的Tree Shaking机制，该部分产物从代码的角度已经是精简过的，要想继续精简只能从业务的角度去分析。

Flutter资源中占比较多的一般是图片，对于图片可以根据业务场景，适当降低图片分辨率，或者考虑替换为网络图片。

**2.压缩法**

因为无论是Android还是iOS，安装包本身已经是压缩包了，对Flutter产物再次压缩的收益很低，所以该方法并不适用。

**3.动态下发**

对于静态资源，理论上是Android和iOS都可以做到动态下发。而对于代码逻辑部分的编译产物，在Android平台支持可执行产物的动态加载，iOS平台则不允许执行动态下发的机器指令。

经过上面的分析可以发现，除了删减、压缩，对所有业务适用、可行且收益明显的进一步优化空间重点在于动态下发了。能够动态下发的部分越多，包大小的收益越大。因此我们决定从动态下发入手来设计一套Flutter包大小优化方案。

## 三、基于动态下发的Flutter包大小优化方案

我们在Android和iOS上实现的包大小优化方案有所不同，区别在于Android侧可以做到so和Flutter资源的全部动态下发，而iOS侧由于系统限制无法动态下发可执行产物，所以需要对产物的组成和其加载逻辑进行分析，将其中非必须和动态链接库一起加载的部分进行动态下发、运行时加载。

当将产物动态下发后，还需要对引擎的初始化流程做修改，这样才能保证产物的正常加载。由于两端技术栈的不同，在很多具体实现上都采用了不同的方式，下面就分别来介绍下两端的方案。

### 3.1 iOS侧方案

在iOS平台上，由于系统的限制无法实现在运行时加载并运行可执行文件，而在上文产物介绍中可以看到，占比较高的App及Flutter这两个均是可执行文件，理论上是不能进行动态下发的，实际上对于Flutter可执行文件我们能做的确实不多，但对于App这个可执行文件，其内部组成的四个模块并不是在链接时都必须存在的，可以考虑部分移出，进而来实现包体积的缩减。

因此，在该部分我们首先介绍Flutter产物的生成和加载的流程，通过对流程细节的分析来挖掘出产物可以被拆分出动态下发的部分，然后基于实现原理来设计实现工程化的方案。

**3.1.1 实现原理简析**

为了实现App的拆分，我们需要了解下App.framework是怎样生成以及各部分资源时如何加载的。如下图4所示，Dart代码会使用gen_snapshot工具来编译成.S文件，然后通过xcrun工具来进行汇编和链接最终生成App.framework。其中gen_snapshot是Dart编译器，采用了Tree Shaking等技术，用于生成汇编形式的机器代码。

![图4 App.framework生成流程示意图](https://p1.meituan.net/travelcube/758d223b258bac30551a6dba93c0052320364.png@745w_158h_80q)

图4 App.framework生成流程示意图



产物加载流程：

![图5 Flutter产物加载流程图](https://p0.meituan.net/travelcube/f5a4ee81491569cefbf0ee35d81e737416951.png@772w_84h_80q)

图5 Flutter产物加载流程图



如上图5所示，Flutter engine在初始化时会从根据 FlutterDartProject 的settings中配置资源路径来加载可执行文件（App）、flutter_assets等资源，具体settings的相关配置如下：

```
// settings
{
...
  // snapshot 文件地址或内存地址
  std::string vm_snapshot_data_path;  
  MappingCallback vm_snapshot_data;
  std::string vm_snapshot_instr_path;  
  MappingCallback vm_snapshot_instr;

  std::string isolate_snapshot_data_path;  
  MappingCallback isolate_snapshot_data;
  std::string isolate_snapshot_instr_path;  
  MappingCallback isolate_snapshot_instr;

  // library 模式下的lib文件路径
  std::string application_library_path;
  // icudlt.dat 文件路径
  std::string icu_data_path;
  // flutter_assets 资源文件夹路径
  std::string assets_path;
  // 
...
}
```

以加载vm_snapshot_data为例，它的加载逻辑如下：

```
std::unique_ptr<DartSnapshotBuffer> ResolveVMData(const Settings& settings) {
  // 从 settings.vm_snapshot_data 中取
  if (settings.vm_snapshot_data) {
    ...
  }
  
  // 从 settings.vm_snapshot_data_path 中取
  if (settings.vm_snapshot_data_path.size() > 0) {
    ...
  }
  // 从 settings.application_library_path 中取
  if (settings.application_library_path.size() > 0) {
    ...
  }

  auto loaded_process = fml::NativeLibrary::CreateForCurrentProcess();
  // 根据 kVMDataSymbol 从native library中加载
  return DartSnapshotBuffer::CreateWithSymbolInLibrary(
      loaded_process, DartSnapshot::kVMDataSymbol);
}
```

对于iOS来说，它默认会根据kVMDataSymbol来从App中加载对应资源，而其实settings是给提供了通过path的方式来加载资源和snapshot入口，那么对于 flutter_assets、icudtl.dat这些静态资源，我们完全可以将其移出托管到服务端，然后动态下发。

而由于iOS系统的限制，整个App可执行文件则不可以动态下发，但在第二部分的介绍中我们了解到，其实App是由kDartVmSnapshotData、kDartVmSnapshotInstructions、kDartIsolateSnapshotData、kDartIsolateSnapshotInstructions等四个部分组成的，其中kDartIsolateSnapshotInstructions、kDartVmSnapshotInstructions为指令段，不可通过动态下发的方式来加载，而kDartIsolateSnapshotData、kDartVmSnapshotData为数据段，它们在加载时不存在限制。

到这里，其实我们就可以得到iOS侧Flutter包大小的优化方案：将flutter_assets、icudtl.dat等静态资源及kDartVmSnapshotData、kDartIsolateSnapshotData两部分在编译时拆分出去，通过动态下发的方式来实现包大小的缩减。但此方案有个问题，kDartVmSnapshotData、kDartIsolateSnapshotData是在编译时就写入到App中了，如何实现自动化地把此部分拆分出去是一个待解决的问题。为了解决此问题，我们需要先了解kDartVmSnapshotData、kDartIsolateSnapshotData的写入时机。接下来，我们通过下图6来简单地介绍一下该过程：

![图6 Flutter Data段写入时序图](https://p0.meituan.net/travelcube/7afa69ddcd7481336d25fe55606b6c5d42295.png@573w_648h_80q)

图6 Flutter Data段写入时序图



代码通过gen_snapshot工具来进行编译，它的入口在gen_snapshot.cc文件，通过初始化、预编译等过程，最终调用Dart_CreateAppAOTSnapshotAsAssembly方法来写入snapshot。因此，我们可以通过修改此流程，在写入snapshot时只将instructions写入，而将data重定向输入到文件，即可实现 kDartVmSnapshotData、kDartIsolateSnapshotData与App的分离。此部分流程示意图如下图7所示：

![图7 Flutter产物拆分流程示意图](https://p0.meituan.net/travelcube/85dc57aa359106c74c9c8691ae9b28d488593.png@1294w_403h_80q)

图7 Flutter产物拆分流程示意图



**3.1.2 工程化方案**

在完成了App数据段与代码段分离的工作后，我们就可以将数据段及资源文件通过动态下发、运行时加载的方式来实现包体积的缩减。由此思路衍生的iOS侧整体方案的架构如下图8所示；其中定制编译产物阶段主要负责定制Flutter engine及Flutter SDK，以便完成产物的“瘦身”工作；发布集成阶段则为产物的发布和工程集成提供了一套标准化、自动化的解决方案；而运行阶段的使命是保证“瘦身”的资源在engine启动的时候能被安全稳定地加载。

![图8 架构设计](https://p0.meituan.net/travelcube/45992e90011ea7d1fc768664f3b8f67c128662.png@1016w_635h_80q)

图8 架构设计



注：图例中MTFlutterRoute为Flutter路由容器，MWS指的是美团云。

**3.1.2.1 定制编译产物阶段**

虽然我们不能把App.framework及Flutter.framework通过动态下发的方式完全拆分出去，但可以剥离出部分非安装时必须的产物资源，通过动态下发的方式来达到Flutter包体积缩减的目的，因此在该阶段主要工作包括三部分。

**1.新增编译command**

在将Flutter包瘦身工程化时，我们必须保证现有的流程的编译规则不会被影响，需要考虑以下两点：

- 增加编译“瘦身”的Flutter产物构建模式, 该模式应能编译出AOT模式下的瘦身产物。
- 不对常规的编译模式（debug、profile、release）引入影响。

对于iOS平台来说，AOT模式Flutter产物编译的关键工作流程图如下图9所示。runCommand会将编译所需参数及环境变量封装传递给编译后端（gen_snapshot负责此部分工作），进而完成产物的编译工作：

![图9 AOT模式Flutter产物编译的关键工作流程图](https://p0.meituan.net/travelcube/d51b6dc367e3391da9487eb11834463a29122.png@937w_426h_80q)

图9 AOT模式Flutter产物编译的关键工作流程图



为了实现“瘦身”的工作流，工具链在图9的流程中新增了buildwithoutdata的编译command，该命令针对通过传递相应参数（without-data=true）给到编译后端（gen_snapshot），为后续编译出剥离data段提供支撑：

```Shell
if [[ $# == 0 ]]; then
  # Backwards-compatibility: if no args are provided, build.
  BuildApp
else
  case $1 in
    "build")
      BuildApp ;;
    "buildWithoutData")
      BuildAppWithoutData ;;
    "thin")
      ThinAppFrameworks ;;
    "embed")
      EmbedFlutterFrameworks ;;
  esac
fi
```

------

```
..addFlag('without-data',
        negatable: false,
        defaultsTo: false,
        hide: true,
  )
```

**2.编译后端定制**

该部分主要对gen_snapshot工具进行定制，当gen_snapshot工具在接收到Dart层传来的“瘦身”命令时，会解析参数并执行我们定制的方法Dart_CreateAppAOTSnapshotAsAssembly，该部分主要做了两件事：

1. 定制产物编译过程，生成剥离data段的编译产物。
2. 重定向data段到文件中，以便后续进行使用。

具体到处理的细节，首先我们需要在gen_sanpshot的入口处理传参，并指定重定向data文件的地址：

```
  CreateAndWritePrecompiledSnapshot() {
    ...
    if (snapshot_kind == kAppAOTAssembly) { // 常规release模式下产物的编译流程
      ...
    } else if (snapshot_kind == kAppAOTAssemblyDropData) { 
      ...
      result = Dart_CreateAppAOTSnapshotAsAssembly(StreamingWriteCallback, 
                                                   file, 
                                                   &vm_snapshot_data_buffer,
                                                   &vm_snapshot_data_size,
                                                   &isolate_snapshot_data_buffer,
                                                   &isolate_snapshot_data_size,
                                                   true); // 定制产物编译过程，生成剥离data段的编译产物snapshot_assembly.S
      ...
    } else if (...) {
      ...
    }
    ...
  }
```

在接受到编译“瘦身”模式的命令后，将会调用定制的FullSnapshotWriter类来实现Snapshot_assembly.S的生成，该类会将原有编译过程中vm_snapshot_data、isolate_snapshot_data的写入过程改写成缓存到buff中，以便后续写入到独立的文件中：

```
// drop_data=true, 表示后瘦身模式的编译过程
// vm_snapshot_data_buffer、isolate_snapshot_data_buffer用于保存 vm_snapshot_data、isolate_snapshot_data以便后续写入文件
Dart_CreateAppAOTSnapshotAsAssembly(Dart_StreamingWriteCallback callback,
                                    void* callback_data, 
                                    bool drop_data,
                                    uint8_t** vm_snapshot_data_buffer,
                                    uint8_t** isolate_snapshot_data_buffer) {
  ...
  FullSnapshotWriter writer(Snapshot::kFullAOT, &vm_snapshot_data_buffer,
                            &isolate_snapshot_data_buffer, ApiReallocate,
                            &image_writer, &image_writer);

  if (drop_data) {
    writer.WriteFullSnapshotWithoutData(); // 分离出数据段
  } else {
    writer.WriteFullSnapshot();
  }
  ...
}
```

当data段被缓存到buffer中后，便可以使用gen_snapshot提供的文件写入的方法 WriteFile来实现数据段以文件形式从编译产物中分离：

```
static void WriteFile(const char* filename, const uint8_t* buffer, const intptr_t size);
// 写data到指定文件中
{
  ...
      WriteFile(vm_snapshot_data_filename, vm_snapshot_data_buffer, vm_snapshot_data_size); // 写入vm_snapshot_data
      WriteFile(isolate_snapshot_data_filename, isolate_snapshot_data_buffer, isolate_snapshot_data_size); // 写入isolate_snapshot_data
  ...
}
```

**3.engine定制**

**编译参数修改**

iOS侧使用-0z参数可以获得包体积缩减的收益（大约为700KB左右的收益），但会有相应的性能损耗，因此该部分作为一个可选项提供给业务方，工具链提供相应版本的Flutter engine的定制。

**资源加载方式定制**

对于engine的定制，主要围绕如何“手动”引入拆分出的资源来展开，好在engine提供了settings接口让我们可以实现自定义引入文件的path，因此我们需要做的就是对Flutter engine初始化的过程进行相应改造：

```
/**
 * custom icudtl.dat path
 */
@property(nonatomic, copy) NSString* icuDataPath;

/**
 * custom flutter_assets path
 */
@property(nonatomic, copy) NSString* assetPath;

/**
 * custom isolate_snapshot_data path
 */
@property(nonatomic, copy) NSString* isolateSnapshotDataPath;

/**
 *custom vm_snapshot_data path
 */
@property(nonatomic, copy) NSString* vmSnapshotDataPath;
```

在运行时“手动”配置上述路径，并结合上述参数初始化FlutterDartProject，从而达到engine启动时从配置路径加载相应资源的目的。

**engine编译自动化**

在完成engine的定制和改造后，还需要手动编译一下engine源码，生成各平台、架构、模式下的产物，并将其集成到Flutter SDK中，为了让引擎定制的流程标准化、自动化，MTFlutter工具链提供了一套engine自动化编译发布的工具。如流程图10所示，在完成engine代码的自定义修改之后，工具链会根据engine的patch code编译出各平台、架构及不同模式下的engine产物，然后自动上传到美团云上，在开发和打包时只需要通简单的命令，即可安装和使用定制后的Flutter engine：

![图10 Flutter engine自动化编译发布流程](https://p1.meituan.net/travelcube/0f7d4b2b186b7a872df1452e4d62f25329888.png@747w_179h_80q)

图10 Flutter engine自动化编译发布流程



**3.1.2.2 发布集成阶段**

当完成Dart代码编译产物的定制后，我们下一步要做的就是改造MTFlutter工具链现有的产物发布流程，支持打出“瘦身”模式的产物，并将瘦身模式下的产物进行合理的组织、封装、托管以方便产物的集成。从工具链的视角来看，该部分的流程示如下图11所示：

![图11 Flutter产物发布集成流程示意图](https://p1.meituan.net/travelcube/e7801120ad3bfc667110803c52077ac846501.png@932w_399h_80q)

图11 Flutter产物发布集成流程示意图



**自动化发布与版本管理**

MTFlutter工具链将“瘦身”集成到产物发布的流水线中，新增一种thin模式下的产物，在iOS侧该产物包括release模式下瘦身后的App.framework、Flutter.framework以及拆分出的数据、资源等文件。当开发者提交了代码并使用Talos（美团内部前端持续交付平台）触发Flutter打包时，CI工具会自动打出瘦身的产物包及需要运行时下载的资源包、生成产物相关信息的校验文件并自动上传到美团云上。对于产物资源的版本管理，我们则复用了美团云提供资源管理的能力。在美团云上，产物资源以文件目录的形式来实现各版本资源的相互隔离，同时对“瘦身”资源单独开一个bucket进行单独管理，在集成产物时，集成插件只需根据当前产物module的名称及版本号便可获取对应的产物。

**自动化集成**

针对瘦身模式MTFlutter工具链对集成插件也进行了相应的改造，如下图12所示。我们对Flutter集成插件进行了修改，在原有的产物集成模式的基础上新增一种thin模式，该模式在表现形式与原有的debug、release、profile类似，区别在于：为了方便开发人员调试，该模式会依据当前工程的buildconfigration来做相应的处理，即在debug模式下集成原有的debug产物，而在release模式下才集成“瘦身”产物包。

![图12 Flutter iOS端集成插件修改](https://p0.meituan.net/travelcube/57a286cda49f10cb5797f12b8d0f7cea162332.png@709w_838h_80q)

图12 Flutter iOS端集成插件修改



**3.1.2.3 运行阶段**

运行阶段所处理的核心问题包括资源下载、缓存、解压、加载及异常监控等。一个典型的瘦身模式下的engine启动的过程如图13所示。

该过程包括：

- **资源下载**：读取工程配置文件，得到当前Flutter module的版本，并查询和下载远程资源。
- **资源解压和校验**：对下载资源进行完整性校验，校验完成则进行解压和本地缓存。
- **启动engine**：在engine启动时加载下载的资源。
- **监控和异常处理**：对整个流程可能出现的异常情况进行处理，相关数据情况进行监控上报。

![图13 iOS侧瘦身模式下engine启动流程图](https://p0.meituan.net/travelcube/db0d00436b2dd6476be3b0f9e014147152136.png@680w_642h_80q)

图13 iOS侧瘦身模式下engine启动流程图



为了方便业务方的使用、减少其接入成本，MTFlutter将该部分工作集成至MTFlutterRoute中，业务方仅需引入MTFlutterRoute即可将“瘦身”功能接入到项目中。

### 3.2 Android侧方案

**3.2.1 整体架构**

在Android侧，我们做到了除Java代码外的所有Flutter产物都动态下发。完整的优化方案概括来说就是：动态下发+自定义引擎初始化+自定义资源加载。方案整体分为打包阶段和运行阶段，打包阶段会将Flutter产物移除并生成瘦身的APK，运行阶段则完成产物下载、自定义引擎初始化及资源加载。其中产物的上传和下载由DynLoader完成，这是由美团平台迭代工程组提供的一套so与assets的动态下发框架，它包括编译时和运行时两部分的操作：

1. 工程配置：配置需要上传的so和assets文件。
2. App打包时，会将配置1中的文件压缩上传到动态发布系统，并从APK中移除。
3. App每次启动时，向动态发布系统发起请求，请求需要下载的压缩包，然后下载到本地并解压，如果本地已经存在了，则不进行下载。

我们在DynLoader的基础上，通过对Flutter引擎初始化及资源加载流程进行定制，设计了整体的Flutter包大小优化方案：

![图14 Android侧Flutter包大小优化方案整体架构](https://p0.meituan.net/travelcube/ba0fba3d447ef081705f655166f81f8454975.png@747w_287h_80q)

图14 Android侧Flutter包大小优化方案整体架构



**打包阶段**：我们在原有的APK打包流程中，加入一些自定义的gradle plugin来对Flutter产物进行处理。在预处理流程，我们将一些无用的资源文件移除，然后将flutter_assets中的文件打包为bundle.zip。然后通过DynLoader提供的上传插件将libflutter.so、libapp.so和flutter_assets/bundle.zip从APK中移除，并上传到动态发布系统托管。其中对于多架构的so，我们通过在build.gradle中增加abiFilters进行过滤，只保留单架构的so。最终打包出来的APK即为瘦身后的APK。

不经处理的话，瘦身后的APK一进到Flutter页面肯定会报错，因为此时so和flutter_assets可能都还没下载下来，即使已经下载下来，其位置也发生了改变，再使用原来的加载方式肯定会找不到。所以我们在运行阶段需要做一些特殊处理：

**1.Flutter路由拦截**

首先要使用Flutter路由拦截器，在进到Flutter页面之前，要确保so和flutter_assets都已经下载完成，如果没有下载完，则显示loading弹窗，然后调用DynLoader的方法去异步下载。当下载完成后，再执行原来的跳转逻辑。

**2.自定义引擎初始化**

第一次进到Flutter页面，需要先初始化Flutter引擎，其中主要是将libflutter.so和libapp.so的路径改为动态下发的路径。另外还需要将flutter_assets/bundle.zip进行解压。

**3.自定义资源加载**

当引擎初始化完成后，开始执行Dart代码的逻辑。此时肯定会遇到资源加载，比如字体或者图片。原有的资源加载器是通过method channel调用AssetManager的方法，从APK中的assets中进行加载，我们需要改成从动态下发的路径中加载。

下面我们详细介绍下某些部分的具体实现。

**3.2.2 自定义引擎初始化**

原有的Flutter引擎初始化由FlutterMain类的两个方法完成，分别为startInitialization和ensureInitializationComplete，一般在Application初始化时调用startInitialization（懒加载模式会延迟到启动Flutter页面时再调用），然后在Flutter页面启动时调用ensureInitializationComplete确保初始化的完成。

![图15 Android侧Flutter引擎初始化流程图](https://p0.meituan.net/travelcube/50c567d29f0b7cc163f4d7bddf448f8953417.png@500w_775h_80q)

图15 Android侧Flutter引擎初始化流程图



在startInitialization方法中，会加载libflutter.so，在ensureInitializationComplete中会构建shellArgs参数，然后将shellArgs传给FlutterJNI.nativeInit方法，由jni侧完成引擎的初始化。其中shellArgs中有个参数AOT_SHARED_LIBRARY_NAME可以用来指定libapp.so的路径。

自定义引擎初始化，主要要修改两个地方，一个是System.loadLibrary(“flutter”)，一个是shellArgs中libapp.so的路径。有两种办法可以做到：

1. 直接修改FlutterMain的源码，这种方式简单直接，但是需要修改引擎并重新打包，业务方也需要使用定制的引擎才可以。
2. 继承FlutterMain类，重写startInitialization和ensureInitializationComplete的逻辑，让业务方使用我们的自定义类来初始化引擎。当自定义类完成引擎的初始化后，通过反射的方式修改sSettings和sInitialized，从而使得原有的初始化逻辑不再执行。

本文使用第二种方式，需要在FlutterActivity的onCreate方法中首先调用自定义的引擎初始化方法，然后再调用super的onCreate方法。

**3.2.3 自定义资源加载**

Flutter中的资源加载由一组类完成，根据数据源的不同分为了网络资源加载和本地资源加载，其类图如下：

![图16 Flutter 资源加载相关类图](https://p0.meituan.net/travelcube/9eb9fa1dcfebf630e4acba5b622c6f6011702.png@428w_291h_80q)

图16 Flutter 资源加载相关类图



AssetBundle为资源加载的抽象类，网络资源由NetworkAssetBundle加载，打包到Apk中的资源由PlatformAssetBundle加载。

PlatformAssetBundle通过channel调用，最终由AssetManager去完成资源的加载并返回给Dart层。

我们无法修改PlatformAssetBundle原有的资源加载逻辑，但是我们可以自定义一个资源加载器对其进行替换：在widget树的顶层通过DefaultAssetBundle注入。

自定义的资源加载器DynamicPlatformAssetBundle，通过channel调用，最终从动态下发的flutter_assets中加载资源。

**3.2.4 字体动态加载**

字体属于一种特殊的资源，其有两种加载方式：

1. **静态加载**：在pubspec.yaml文件中声明的字体及为静态加载，当引擎初始化的时候，会自动从AssetManager中加载静态注册的字体资源。
2. **动态加载**：Flutter提供了FontLoader类来完成字体的动态加载。

当资源动态下发后，assets中已经没有字体文件了，所以静态加载会失败，我们需要改为动态加载。

**3.2.5 运行时代码组织结构**

整个方案的运行时部分涉及多个功能模块，包括产物下载、引擎初始化、资源加载和字体加载，既有Native侧的逻辑，也有Dart侧的逻辑。如何将这些模块合理的加以整合呢？平台团队的同学给了很好的答案，并将其实现为一个Flutter Plugin：flutter_dynamic（美团内部库）。其整体分为Dart侧和Android侧两部分，Dart侧提供字体和资源加载方法，方法内部通过method channel调到Android侧，在Android侧基于DynLoader提供的接口实现产物下载和资源加载的逻辑。

![图17 FlutterDynamic结构图](https://p0.meituan.net/travelcube/29e8a2ed72c317d1978a7da8d9c64f7324388.png@558w_444h_80q)

图17 FlutterDynamic结构图



## 四、方案的接入与使用

为了让大家了解上述方案使用层面的设计，我们在此把美团内部的使用方式介绍给大家，其中会涉及到一些内部工具细节我们暂不展开，重点解释设计和使用体验部分。由于Android和iOS的实现方案有所区别，故在接入方式相应的也会有些差异，下面针对不同平台分开来介绍：

### 4.1 iOS

在上文方案的设计中，我们介绍到包瘦身功能已经集成进入美团内部MTFlutter工具链中，因此当业务方在使用了MTFlutter后只需简单的几步配置便可实现包瘦身功能的接入。iOS的接入使用上总体分为三步：

1.引入Flutter集成插件（cocoapods-flutter-plugin 美团内部Cocoapods插件，进一步封装Flutter模块引入，使之更加清晰便捷）：

```
gem 'cocoapods-flutter-plugin', '~> 1.2.0'
```

2.接入MTFlutterRoute混合业务容器（美团内部pod库，封装了Flutter初始化及全局路由等能力），实现基于“瘦身”产物的初始化：

Flutter 业务工程中引入 mt_flutter_route：

```
dependencies:
  mt_flutter_route: ^2.4.0
```

3.在iOS Native工程中引入MTFlutterRoute pod：

```
binary_pod 'MTFlutterRoute', '2.4.1.8'
```

经过上面的配置后，正常Flutter业务发版时就会自动产生“瘦身”后的产物，此时只需在工程中配置瘦身模式即可完成接入：

```
flutter 'your_flutter_project', 'x.x.x', :thin => true
```

### 4.2 Android

**4.2.1 Flutter侧修改**

1. 在Flutter工程pubspec.yaml中添加flutter_dynamic（美团内部Flutter Plugin，负责Dart侧的字体、资源加载）依赖。
2. 在main.dart中添加字体动态加载逻辑，并替换默认资源加载器。

```
void main() async {
   // 动态加载字体
  await dynFontInit();
  // 自定义资源加载器
  runApp(DefaultAssetBundle(
    bundle: dynRootBundle,
    child: MyApp(),
  ));
}
```

**4.2.2 Native侧修改**

1.打包脚本修改

在App模块的build.gradle中通过apply特定plugin完成产物的删减、压缩以及上传。

2.在Application的onCreate方法中初始化FlutterDynamic。

3.添加Flutter页面跳转拦截。

在跳转到Flutter页面之前，需要使用FlutterDynamic提供的接口来确保产物已经下载完成，在下载成功的回调中来执行真正的跳转逻辑。

```
class FlutterRouteUtil {
    public static void startFlutterActivity(final Context context, Intent intent) {
        FlutterDynamic.getInstance().ensureLoaded(context, new LoadCallback() {
            @Override
            public void onSuccess() {
              // 在下载成功的回调中执行跳转逻辑
                context.startActivity(intent);
            }
        });
    }
}
```

备注：如果App有使用类似[WMRoute](https://github.com/meituan/WMRouter)之类的路由组件的话，可以自定义一个UriHandler来统一处理所有的Flutter页面跳转，同样在ensureLoaded方法回调中执行真正的跳转逻辑。

4.添加引擎初始化逻辑

我们需要重写FlutterActivity的onCreate方法，在super.onCreate之前先执行自定义的引擎初始化逻辑。

```
public class MainFlutterActivity extends FlutterActivity {
    @Override
    protected void onCreate(Bundle savedInstanceState) 
      // 确保自定义引擎初始化完成
        FlutterDynamic.getInstance().ensureFlutterInit(this);
        super.onCreate(savedInstanceState);
    }
}
```

## 五、总结展望

目前，动态下发的方案已在美团内部App上线使用，Android包瘦身效果到达95%，iOS包瘦身效果达到30%+。动态下发的方案虽然能显著减少Flutter的包体积，但其收益是通过运行时下载的方式置换回来的。当Flutter业务的不断迭代增长时，Flutter产物包也会随之不断变大，最终导致需下载的产物变大，也会对下载成功率带来压力。后续，我们还会探索Flutter的分包逻辑，通过将不同的业务模块拆分来降低单个产物包的大小，来进一步保障包瘦身功能的可用性。