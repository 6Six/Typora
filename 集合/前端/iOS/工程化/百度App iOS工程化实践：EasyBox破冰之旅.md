**前言**

百度App从单一的搜索工具发展到今天以搜索和Feed流为双引擎的综合性内容消费服务平台，其复杂程度已然不可同日而语矣。 作为一个日活过亿的超级App，业务规模庞大，相关技术人员超过千人，客户端支持主流的移动技术，涉及近百业务方，技术形态复杂，各种组件近三百个，代码百万量级，由此带来的工程化问题是技术团队的一个极大挑战。

项目的膨胀导致了很多不起眼的小问题被无限放大，组件管理不规范、编译时间长、工程文件合并冲突、Xcode默认非彻底编译隔离等等问题，导致开发人员在开发环境上耗费了大量时间。目前业界较流行的工具对于大规模工程的支持力度相对较弱，实践起来总是有些掣肘，难以达到理想状态。

EasyBox的诞生，就是致力于为超级App量身打造一套现代、高效、优雅的研发工具链。

这篇文章的主要目的是**站在工具链的角度**上，分享一下我们在实践工程化过程中一些经验。



**概述**

EasyBox主体由工程组装器(Installer)、多仓库管理工具(MGit)、二进制管理工具(LFS)三部分构成，分别负责工作区的构建(组件依赖分析、工程的生成与组合)、源码仓库的管理以及二进制的管理。

![图片](https://mmbiz.qpic.cn/mmbiz_png/Ce6bSqXkduwmGNKchsIDSq7bicr1qotmtYcg5btolDLnBpZKDESlrarwjkaYQjD3MZKOZek9E82IYQogGSr4Akw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

EasyBox架构图

由多仓库管理工具克隆所需仓库源码，由二进制管理工具下载二进制包，然后组装器根据描述表生成对应工程，组合层级并最终生成Workspace。

![图片](https://mmbiz.qpic.cn/mmbiz_png/Ce6bSqXkduwmGNKchsIDSq7bicr1qotmtfF0uSl6fTNejiaHWqyrTzpJicZcTCa5QkMic8nG0coHnSCRDN5icfIek6g/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

简化工作流程示意图

**实践**

**—****—**

**EasyBox诞生的过程本质上是工程化逐步深入的过程。**分而治之的道理大家都明白，但这并不意味着工程化就是简单的拆库重组。其目的是要对项目工程进行合理化改造，让开发人员能够快速理解工程架构并进入开发状态，避免开发人员在开发环境上花费过多时间，从而提高编码、测试等阶段的研发效率。在这个过程中我们在**规范组件管理与使用、强化工程能力、提升编译速度**这三个方面不断优化，最终形成EasyBox独特的优势。

**丨1. 规范组件管理与使用**

组件(源码/二进制)统一使用描述表(boxspec)描述，依赖与API管理均由描述表唯一决定，配合编译隔离使得组件边界划分明确。组件的版本号严格遵守语义化版本(Semantic Versioning)规范，组件新版本发布时，需经过持续集成分析API变化等一系列检查之后方可完成发布。

**破窗理论** 环境中的不良现象如果被放任存在，会诱使人们仿效，甚至变本加厉。

**组件基于源码发布来完成二进制发布及接口发布**，以确保安全性，方便进行源码回归校验以及必要时基于同一节点发布不同类型的二进制，接口发布用于完成矩阵产品的组件替换。发布版本的描述表与二进制文件交由专门的文件服务器管理，并禁止二进制文件入源码仓库以避免仓库膨胀。更多源码与二进制管理细节参见后文介绍。

**丨2. 强化工程能力**

**2.1 组件的独立与隔离**

与业界其他大多数工具不同，EasyBox采取了**每个组件都拆分为独立工程**的方案，同时也将不同的组件的编译产物放在了不同的目录下，这样做的目的是**确保彻底的编译隔离**。

这里其实是源自Xcode的遗留的两个坑：

- 同一个工程的OC/C/C++文件就算不在同一个Target下且没有依赖关系也可以互相访问(Swift是不允许的)。
- Xcode会自动将编译产物所在的文件夹(BUILD_PRODUCTS_DIR)添加到 FRAMEWORK_SEARCH_PATHS 中，而同一个Workspace下的编译产物默认是在同一文件夹下的。

这两个坑都会打破编译隔离，后者更会导致在不同开发者的电脑上编译结果不同，时常出现自己编译过了而别人编译不过的情况，这是其实由于组件编译次序不同导致的。

但由于Xcode工程文件一直是饱受诟病的设计，多人协作时工程文件合并冲突简直就是一场噩梦，所以我们采用了组件配置表的方案来完成**去工程文件化**，每个组件通过配置一个boxspec文件(类似于CocoaPods的podspec)，组件源码会根据实际目录映射对应工程结构，同时生成一个xcfilelist来维护当前组件所配置的资源列表，并最终生成工程文件。该工程文件不被Git追踪，从而避免合并冲突问题。

编译隔离可以带来很多好处，比如组件边界一定是明确的，几百个组件组合在一起能不能编译过不再看"运气"。它限制了组件修改时所影响的范围，这将有助于组件在不同环境(App)下编译构建、多端复用，使得输出时开发者对于组件的依赖是有预期的。

另外配合接口发布及组件化中的协议解耦，使得组件输出时可以选择性的只携带其他组件的接口(而非实现)，从而很容易地完成依赖组件的剥离与替换。

![图片](https://mmbiz.qpic.cn/mmbiz_png/Ce6bSqXkduwmGNKchsIDSq7bicr1qotmttA4OHiafibQ6D61eHuLSGMeT3EMp8Y22zrwDOOJFUJI7w7PjvynXgtZQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

boxspec描述表示例

![图片](https://mmbiz.qpic.cn/mmbiz_png/Ce6bSqXkduwmGNKchsIDSq7bicr1qotmto9VgLnjqbAWPXBGiaU3DiccY0EjYHaZyGK9jicxeXWWgOQD99wfVSQfqg/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

自动生成组件工程示例

**2.2 多仓库的拆分与管理**

代码集中管理导致主仓库越来越臃肿，而且代码权限问题难以管理，这对于大团队的开发而言是个非常痛苦的事情，很容易出现组件在负责人不知道的情况被其他人合入代码，加之一定的组件易手率，情况会变得更加糟糕。

**熵增原理** 在自然情况下，一切物质都将趋向于无序。

为解决这个问题，我们将各组件拆分为独立仓库，**完成物理隔离，做到入库权限收紧，物各有主**。另外这对于多产品复用起了至关重要的作用，也是中台化建设的必要条件。在实际实践过程中，为了避免某些仓库过于琐碎，也会出现一个仓库多个组件的情况，但并不影响大局。

多仓库的拆分从某种程度上加剧了工程的复杂度，开发者不可避免地出现需要操作多个仓库的情况，这时直接使用Git操作成本高且易出错，为此我们专门设计了多仓库管理工具(MGit, Android/iOS双端共用)。与Android系统源码多仓库管理工具Repo不同，**MGit保持了Git的大多数指令和用法，****同时在内部执行时保证多仓库操作的安全性，在执行风险操作时做出必要提醒**。开发者只需使用 mgit 替代 git 命令即可完成多仓库操作，这样既保持了大多数开发者的使用习惯，又可以安全方便地同时操作多个仓库。同时我们利用Gerrit的topic机制并加以改造，实现多仓库的分组提交以确保原子性入库，进而保证自动打包机制的正常运作。

![图片](https://mmbiz.qpic.cn/mmbiz_png/Ce6bSqXkduwmGNKchsIDSq7bicr1qotmtRzfcj5wbkQJ4INgsaCTTL3PvuafaBhy2tWPKJFn4mBRAyXl1FEduQA/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

MGit与Repo指令对比图

![图片](https://mmbiz.qpic.cn/mmbiz_png/Ce6bSqXkduwmGNKchsIDSq7bicr1qotmtwOiaPK5w9EUHrs0edfr4ia3w9CicKArGgBXf55h1vUgVxV22E8CJvB7fw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

MGit使用示例

多仓库带来一个最核心的问题是**组件节点同步问题**。当不同组件的源码处于不同节点，很容易出现编译不过或功能不正常等问题。可以通过采用**『同名分支原则』**来规避这个问题，即将所有要开发的仓库保持在同一个分支，再配合其他组件的版本依赖管理，一起保证各组件节点的匹配。这个规则非常简单易记，在大团队推广起来也不会有什么成本。

**丨仓库嵌套问题** 

由于历史遗留原因，我们在拆仓库的初期选择了在组件原有位置创建仓库，最初设计上考虑不周给我们带来了不小的麻烦，不同分支的gitignore的不同会引起文件追踪状态的错乱，后来便采用将源码仓库进行平铺，以避免这个问题。

**2.3 层级的动态划分与构建**

大型项目基本都会走上分层架构之路，分层带来的好处也是显而易见的，软件架构更加清晰，规范更容易确立，而层级的单向访问也有助于降低工程复杂度。**分层设计本质上是对开闭原则的践行**，这一原则通常是我们对架构设计时的主导原则之一，其目的是让软件更易于扩展，限制每次修改所影响的范围。具体就是将软件划分为一系列组件，并将这些组件按层级进行组织，使得下层组件不会因为上层组件的修改而受到影响。

所以与其他工具不同的是，EasyBox选择在设计上支持层级的划分，我们希望EasyBox不仅仅承担包管理器的作用，也起到帮助架构师规范好整个项目的作用。随着团队的扩大，依赖不合理的问题更加显著，很多事情不再是简单的给个规范、喊一嗓子就可以解决的，这时我们倾向于制定强硬的规则，来确保不会出现明显的问题。**分层的设计也可以让团队新成员快速理解工程架构设计，时刻提醒系统边界的重要性，同时建立对依赖的约束限制**。

"立法" 要远比 "道德规范" 来的直接有效。

约束的建立可以规避很多问题，举个例子，PM要求对某个视图的展示事件打点，如果对组件化理解不深或者偷懒的话，很容易直接在UI库里直接进行打点，而这将为后续组件的复用带来很大的问题。

![图片](https://mmbiz.qpic.cn/mmbiz_png/Ce6bSqXkduwmGNKchsIDSq7bicr1qotmtMTiac8LNKpLoDicnSlIlNmvDP9QCicDibCUialR9eeDiaITVA0rAAsQEqKjw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

百度App现行架构图

(上层组件可以访问下层组件，不可逆向访问)

![图片](https://mmbiz.qpic.cn/mmbiz_png/Ce6bSqXkduwmGNKchsIDSq7bicr1qotmtOvJfrJFDcqxDSVWJmWBcdKFHc70tCaQ9yAYseSsoG0ITKwu0VJDj4g/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

层级配置示例

![图片](https://mmbiz.qpic.cn/mmbiz_png/Ce6bSqXkduwmGNKchsIDSq7bicr1qotmt06VsdcYEB4icLLhlt6umNib2rrFFGyAg1R3oWt6iaicpPvxWAsvS6WibibXA/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

自动生成工程示例

在组合构建过程中，无论是组件还是子组件，都采用了直接链接到最终产物(App或Dynamic Framework)的做法，其原因是静态库之间的合并风险是很高的，比如符号重复时会仅仅会给出一个警告，然后触发自动裁剪。

**丨force_load问题** 

如果App兼容iOS8的话，苹果对于主包二进制有大小限制，这时可以将底层Layer改为动态库来减少主包二进制体积，此时一些C/C++的组件如果出现跨层调用时是需要force_load的，实际应用过程中应尽可能地使用OC封装这些库而避免上层业务直接使用这些组件，尽可能的少使用force_load，从而避免包体积增大。

**丨3. 提升编译速度**

**3.1 组件二进制化**

业务的膨胀导致百度App代码激增，这大幅拖累了编译速度，仅主业务(抛开几十家业务方及ffmpeg、opencv之类的重量级三方库)编译时长也接近20分钟(13' RMBP)，所以我们决定采用二进制化方案来解决这个问题，即由集群将组件打包为二进制并上传至文件服务器，开发时**仅保留需要开发的组件的源码，其他各组件均以二进制存在**。通过二进制化，**正常情况下**(1至3个处于开发模式的组件)**全量编译****时间压缩至2分钟**(13' RMBP)**以内**，增量编译速度也明显加快，而对于工程文件的缓存也可以有效减少全量编译的次数。

宿主工程的配置是由Boxfile、Boxfile.overlay、Boxfile.local三个文件配合完成的，配置生效优先级Boxfile.local > Boxfile.overlay > Boxfile。overlay和local格式相同，都是用于开发联调阶段使用的临时配置文件，用于二进制源码的切换，区别在于overlay被git追踪，用于多人协同开发以及持续集成打包，而local则不被git追踪，仅用于本地调试。当二进制切回开发模式时，如果该仓库分支不存在，会根据当前组件版本节点(而不是master)创建对应分支，以保证分支的起始节点同步。

![图片](https://mmbiz.qpic.cn/mmbiz_png/Ce6bSqXkduwmGNKchsIDSq7bicr1qotmtYjqWVM1TjWWhooBtpNKATO4YXW4Vl7zVtRnncCrwoxzMh5b88ianpgQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

Boxfile.overlay配置示例

![图片](https://mmbiz.qpic.cn/mmbiz_png/Ce6bSqXkduwmGNKchsIDSq7bicr1qotmtceJOX0elM0pkZBMGpZng2nias06AwSf96HukajD4L5ZQDCv1katRqbA/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

可视化配置工具示例

**丨二进制失效问题** 

二进制化对组件接口层的稳定是有很强的要求的，组件接口层应当尽可能稳定，尤其注意宏/枚举等声明改动引起其他组件二进制失效问题，接口层尽可能采用增量扩展的形式(旧接口标记废弃)。此时建设监测机制确保版本号的正确性就显得尤为重要，我们通过Clang插件来完成对API变动的监控，在组件发布之前进行校验。

二进制化还会带来一个很大的问题就是给开发者的调试带来不便。Java、JS等语言都会有很完善的**源码映射**(Source Map)机制来弥补打包后带来的调试问题，而对于OC/Swift来说，这方面的建设却是非常少见的。用过Carthage的同学都知道，Carthage可以从工程单步调试进入到源码里面的，但是这仅局限于本地编译出来的二进制，而且也只能从外部通过单步调试进入。而我们希望达到的效果是：

二进制包由集群编译打包，本地开发时使用二进制文件，工程根据配置自动导入源码完成源码映射，源码不参与编译，但断点调试依然有效。

这里要先理解断点的本质是什么，断点其实是一个含有触发条件的坐标，**断点 = 源文件位置 + 代码行数 + 触发条件**。

lldb下通过 breakpoint list 查看断点信息

![图片](https://mmbiz.qpic.cn/mmbiz_png/Ce6bSqXkduwmGNKchsIDSq7bicr1qotmtHVcAFJG7he9PIhtrRDjWFCA02lwCWB7fQnI93NHrd5uVQdSjpOSOOQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

集群编译和本机编译的区别在于源文件的位置，所以只要保证源文件位置相同，就可以达到理想效果。可以借助编译参数 -fdebug-prefix-map 来完成源文件位置的匹配，在集群编译时通过该参数将组件源码目录指向 /tmp/easybox/$(VIRTUAL_ID) (VIRTUAL_ID是根据组件信息与时间戳生成的定长字符串)。当EasyBox需要进行源码映射时，只需导出一份对应时间节点的源码，然后将该源码目录软链到 /tmp/easybox/$(VIRTUAL_ID) 目录下，映射就已经完成了。再将该文件添加至工程中(不参与编译，引用路径须是 /tmp/easybox/$(VIRTUAL_ID)/* )，此时便可以愉快地玩(tiao)耍(shi)了。

![图片](https://mmbiz.qpic.cn/mmbiz_png/Ce6bSqXkduwmGNKchsIDSq7bicr1qotmt557FLIG2I7PiaC1RwXljAymPuf2jpjsBAFRP9S0w81zvyBOsg4ic8PMQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

源码映射原理示意图

我们可以借助 dwarfdump 命令查看二进制中相关的Debug信息

![图片](https://mmbiz.qpic.cn/mmbiz_png/Ce6bSqXkduwmGNKchsIDSq7bicr1qotmtYRMUT3ypP9OpiaHMSuxlKcRYSxK1icerPyV5o8QKcHMuJyujjRzXEl2A/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

修改前后的对比图

**丨其他方案** 

翻阅LLVM文档可以查到另外一些关于源码映射的资料：

\1. 在运行阶段，可以借助lldb的source-map来修改源文件所处的位置，显然使用起来很不方便。

\2. 动态链接库可以通过配置plist文件来完成源码映射，但在实际应用中，动态库会严重影响启动速度和包体积，通常各个组件均以静态库存在，故也未采用此方案。

二进制化之后，编译速度有了质的飞跃，在实践过程中还针对一些细节的进行优化。

**3.2 Clang模块缓存（头文件检索缓存）**

百度App现行有近三百个组件，组件间的依赖关系非常复杂，组件规模大小不一，组件间进行调用时在预处理阶段不同文件反复的import导致的头文件检索过程十分耗时，如苹果推荐，我们将大多数组件编译为framework，而在非framework的情况下，可以通过生成modulemap来完成static library的Clang Module Cache，具体可参见 WWDC 18: Behind the Scenes of the Xcode Build Process，这里就不再赘述。

需要注意的是，Clang Module Cache并不会随着Xcode Clean而被清除，之前偶尔会出现由该Cache引起的一些奇怪的编不过的bug，此时则需要将DerivedData下的ModuleCache.noindex文件夹清除即可，不过随着后来Xcode的更新，这一问题得到了缓解。

**3.3 资源编译缓存**

为了优化包体积，百度App工程大部分图片资源采用的是xcassets，在打包App过程中，需要通过actool将全部的xcassets编译合并进一个car文件，actool在处理时并没有做缓存，由于图片资源较多，每次编译xcassets都耗时近一分钟(13' RMBP)。由于源码增量编译本身就比较快，那这一分钟对于编译速度实际的影响是非常大的。解决办法就是通过rsync备份一份资源来检验xcassets是否发生过变化，并只在资源或条件发生变化再重新触发编译。

**理念**

**—****—**

在实践EasyBox的过程中，我们逐步抽象确立了一些理念，这些理念在我们工程化的过程中起到了重要作用。

**丨法治优先**

项目经过长期的沉淀，往往会形成各种规范，团队规模大、规范多，不再是群里喊一嗓子就可以解决的事情，应该尽可能地通过工具来实行**强制约束**来确保规范被执行。规范也应当尽可能的简单易记，并通过工具辅助将遵守成本降到最低。

**丨限制管理（编译隔离）**

组件边界要明确，组件的依赖关系、API接口等要可控，这将有助于组件在不同环境(App)下编译构建、多端复用，使得输出时开发者对于组件的依赖是有预期的。反之，当组件边界模糊时，真正输出组件时，会暴露诸多问题，很容易造成拔出萝卜带出泥的窘境。

**丨问题前置**

工具能够暴露的问题，应当尽可能早的暴露出来，譬如能在提代码之前暴露的就不要延迟到持续集成暴露，问题发现的越早，修复成本就越低，这有助于避免开发者在实际开发过程中反复返工。

**对比业界**

**—****—**

业内比较广泛采用的是将CocoaPods作为工具链，其对于小工程的实践是非常经典的。但是对大工程的支持力度是相对较弱的，其编译隔离与去工程化是相互矛盾的(最近发布的1.7 beta开始支持独立工程完全隔离了，但还处于非常初期的阶段)，不支持层级划分，对于多仓库、二进制管理也都是不无小补，而这些几乎是大工程实践必备的工程能力。

在技术方案上，EasyBox设计之初的理念就与CocoaPods有着很大的区别，所以并没有采用业界通用做法基于CocoaPods改造，而是直接基于xcodeproj重新实现了一套全新的工具链。

![图片](https://mmbiz.qpic.cn/mmbiz_png/Ce6bSqXkduwmGNKchsIDSq7bicr1qotmtmcq6cSL91bfSlnpNJ58C6oYTBD1e7QQyWrT7uRWrO212psAWEGmDMA/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

功能对比图

**未来**

**—****—**

EasyBox自上线以来自身也在不断地迭代进化，目前还存在一些地方不够完善，后续会将已经适配其他包管理器的三方开源库无缝兼容。目前内部已在百度App、西番、看多多等多个团队得到应用，后续还会推广至更多团队。同时，我们后续也将推动开源工作，将整套解决方案打包开源出去。

**结语**

**—****—**

近年来，国内对超级App的追逐，导致客户端项目极速膨胀，而由此带来的工程化问题变得尤为严重。工欲善其事，必先利其器。大型团队向研发工具链方向的投入往往是用户所看不到的，但是对开发者来说收益却是非常明显的。组件规范统一，维护、换手成本相对较低；完善的物理隔离与编译隔离将组件边界划分清晰，物各有主；层级的引入使得架构层级更加明晰，上手成本更低；在不影响开发人员调试成本的情况下，将编译速度压至2分钟以内(提升90%)，也大幅提升了App出包速度。

我们之前将百度App(iOS端)原有零散的研发工具推倒重来，从零搭建了这套现代、高效、优雅的研发工具链。工具链对于组件多产品复用、中台化建设也起着至关重要的支撑作用。但工具并不能解决代码问题，得益于组件化工作的推进(后续将会有专文介绍)，使得百度App这头大象非常顺利地在短时间内就穿上了这套装甲，让这只步履蹒跚的大象也可以拥有矫健灵活的舞步。

EasyBox更多承担的是开发环境的配置，配合背后强大的流程扭转中枢，将开发与后续的提测、发布、准入等工作流形成标准化研发闭环，组成集管理、迭代、输出、集成等功能于一体的一站式研发中台，感兴趣的同学敬请期待后续文章。

**参考资料**

**—****—**

\1. https://semver.org/

\2. https://llvm.org/docs/SourceLevelDebugging.html

\3. https://lldb.llvm.org/use/symbols.html

\4. https://github.com/apple/swift-package-manager/blob/master/Documentation/Internals/PackageManagerCommunityProposal.md

5.https://developer.apple.com/videos/play/wwdc2018/415/

\6. https://cocoapods.org/

\7. https://gerrit.googlesource.com/git-repo/

\8. https://gerrit-review.googlesource.com/Documentation/intro-user.html#topics