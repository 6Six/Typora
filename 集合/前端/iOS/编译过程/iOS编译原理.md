# [iOS编译原理](https://www.jianshu.com/p/3b20891ae045)



```
内容目录：
1、基础解释
2、iOS设备的CPU架构
3、ARM处理器指令集
4、i386 | x86_64指令集
5、Xcode中指令集
6、编译器LLVM、解释器
```



#### 一般可以将编程语言分为两种，

wiki：[编译语言](https://links.jianshu.com/go?to=https%3A%2F%2Fzh.wikipedia.org%2Fwiki%2F%E7%B7%A8%E8%AD%AF%E8%AA%9E%E8%A8%80)、[直译式语言](https://links.jianshu.com/go?to=https%3A%2F%2Fen.wikipedia.org%2Fwiki%2FInterpreted_language)
 百度：[编译型语言](https://links.jianshu.com/go?to=https%3A%2F%2Fwww.baidu.com%2Flink%3Furl%3DNkK-R2hpRv709irjmKs0LXTXpZYWmM1rt5VpqDgI8CBPqO8ckeAWdWtAnlUPmdj1ZEdrfZYpuooyiffRjhjo5ZEb0UfzYixQz1lwn_nxQt8j6U7yf5woYL0qiy1ZwP29tyPFGhSQT1m5OEJppwsoha%26wd%3D%26eqid%3D8fd23fc1000540dd000000045cbffb72https%3A%2F%2Fbaike.baidu.com%2Fitem%2F%E7%BC%96%E8%AF%91%E5%9E%8B%E8%AF%AD%E8%A8%80%2F9564109%3Ffr%3Daladdin)（编译器处理）、[直译语言](https://links.jianshu.com/go?to=https%3A%2F%2Fbaike.baidu.com%2Fitem%2F%E7%9B%B4%E8%AF%91%E8%AF%AD%E8%A8%80%2F13827258%3Ffr%3Daladdin)（解释器处理）

**解释器**：是在运行时才去解析代码，获取一段代码后就会将其翻译成目标代码（就是字节码：Bytecode），然后一句一句地执行目标代码。解释器可以在运行时去执行代码，说明它具有动态性，程序运行后能够随时通过增加和更新代码来改变程序的逻辑。

**编译器**：把一种编程语言(原始语言)转换为另一种编程语言(目标语言)，编译生成一份完整的机器码再去执行。

###### 大多数编译器由两部分组成：前端和后端。

- 前端：负责词法分析，语法分析，生成中间代码；
- 后端：以中间代码作为输入，进行行架构无关的代码优化，接着针对不同架构生成不同的机器码。

前端/后端 依赖统一格式的 **中间代码(IR)**，使得前后端可以独立的变化。新增一门语言只需要修改前端，而新增一个CPU架构只需要修改后端即可。

Objective C/C/C++使用的编译器前端是[clang](https://links.jianshu.com/go?to=https%3A%2F%2Fclang.llvm.org%2Fdocs%2Findex.html)，swift是[swift](https://links.jianshu.com/go?to=https%3A%2F%2Fswift.org%2Fcompiler-stdlib%2F%23compiler-architecture)，后端都是[LLVM](https://links.jianshu.com/go?to=https%3A%2F%2Fllvm.org%2F)。
 

### **1、基础解释：**

程序编译一般需经几个步骤：预处理、编译、汇编、链接。

**编译**：是将将人类可读的程序代码文本 --> 翻译成为 --> 计算机可以执行的二进制指令机器码。即：源程序 --> 目标程序

**编译器前端** Clang：预处理 --> 词法分析 --> 语法分析 --> 生成IR（Clang Code Generator）

**编译器后端** LLVM：对IR优化 --> 目标代码--> 汇编器 --> 机器码（LLVM Code Generator）--> 链接 --> Mac-O文件

##### **可执行文件**：

(executable file) 指的是可以由操作系统进行加载执行的文件。在不同的操作系统环境下，可执行程序的呈现方式不一样。在windows操作系统下，可执行程序可以是 .exe文件 .sys文件 .com等类型文件。

##### **可执行程序**：

（executable program，EXE File）是可在操作系统[存储空间](https://links.jianshu.com/go?to=https%3A%2F%2Fbaike.baidu.com%2Fitem%2F%E5%AD%98%E5%82%A8%E7%A9%BA%E9%97%B4%2F10657950)中浮动定位的二进制可执行程序。它可以加载到内存中，由[操作系统](https://links.jianshu.com/go?to=https%3A%2F%2Fbaike.baidu.com%2Fitem%2F%E6%93%8D%E4%BD%9C%E7%B3%BB%E7%BB%9F%2F192)加载并执行。特定的CPU[指令集](https://links.jianshu.com/go?to=https%3A%2F%2Fbaike.baidu.com%2Fitem%2F%E6%8C%87%E4%BB%A4%E9%9B%86%2F238130)（如[X86](https://links.jianshu.com/go?to=https%3A%2F%2Fbaike.baidu.com%2Fitem%2FX86%2F6150538)指令集）对应的不同平台之间的可执行程序不可直接移植运行。
 

### 2、iOS 设备的CPU架构

##### 2.1、在模拟器上支持:

模拟器32位处理器 iPhone4s-5：          i386 架构
 模拟器64位处理器 iPhone5s-8 Plus：x86_64 架构
 （ios模拟器没有arm指令集）

##### 2.2、在真机设备上支持:

真机32位处理器：armv7、armv7s
 真机64位处理器：arm64
 armv6：  iPhone、iPhone 2、iPhone 3G、iPod Touch(第一代)、iPod Touch(第二代)
 armv7：  iPhone 3Gs、iPhone 4、iPhone 4s、iPad、iPad 2
 armv7s：iPhone 5、iPhone 5c (静态库只要支持了armv7,就可以在armv7s的架构上运行)
 arm64： iPhone 5s、iPhone 6、iPhone 6 Plus、iPhone 6s、iPhone 6s Plus、iPad Air、iPad Air2、iPad mini2、iPad mini3

### 3、ARM处理器指令集

几乎所有手机处理器都基于ARM处理器的,ARM处理器特点是体积小、低功耗、低成本、高性能，所以在嵌入式系统中应用广泛
 armv6｜armv7｜armv7s｜arm64都是ARM处理器的指令集
 这些指令集都是向下兼容的

### 4、i386｜x86_64 指令集：i386和x86_64 是Mac处理器的指令集

- i386是针对intel通用微处理器32位处理器的
- x86_64是针对x86架构的64位处理器
   所以当使用iOS模拟器的时候会遇到i386｜x86_64（ios模拟器没有arm指令集）

###### 命令查看静态库支持的架构：

通过 lipo -info - 》 拖入模拟器或者真机.framework的路径 查看静态库支持的架构（以下是终端命令）
 $ lipo -info - .framework的路径
 

### 5、Xcode中指令集（相关选项Build Setting中设置）



```undefined
•   Architectures（架构）
•   Valid Architectures（有效架构）
•   Build Active Architecture Only（只构建活动架构）
```

![img](https:////upload-images.jianshu.io/upload_images/2154347-72375f025fc03b74.png?imageMogr2/auto-orient/strip|imageView2/2/w/1146/format/webp)

Pasted Graphic 1.png

###### 5.1. Architectures（架构）

指定工程被编译成支持哪些指令集类型.
 支持的指令集越多，就会编译出很多个指令集代码的数据包，对应生成二进制包就越大，也就是ipa包越大。
 所以现在支持 iPhone5 以上，只需要 arm64 就行了，armv6｜armv7｜armv7s 这三个都可以干掉了。

###### 5.2. Valid Architectures（有效架构）

限制可能被支持指令集的范围.
 xcode编译出来的二进制包类型最终从这些类型产生，而编译出哪些指令集的包，将由Architectures与Valid Architectures这些交集来确定,面会举例说明.

###### 5.3. Build Active Architecture Only（只构建活动架构）

指定只对当前连接设备所支持的指令集编译。
 当设置为YES时是为了debug编译的速度更快，它只会编译当前的architecture版本。
 当设置为NO时，会编译所有的版本，所以一般debug设置为YES，release设置为NO以适应不同设备。
 

### 6.  编译器 LLVM

> 2000年，伊利诺伊大学厄巴纳－香槟分校（简称UIUC）这所享有世界声望的一流公立研究型大学的 Chris Lattner（他的 twitter [@clattner_llvm](https://links.jianshu.com/go?to=https%3A%2F%2Ftwitter.com%2Fclattner_llvm) ） 开发了一个叫作 Low Level Virtual Machine 的编译器开发工具套件，后来涉及范围越来越大，可以用于常规编译器，JIT编译器，汇编器，调试器，静态分析工具等一系列跟编程语言相关的工作，于是就把简称 LLVM 这个简称作为了正式的名字。Chris Lattner 后来又开发了 Clang，使得 LLVM 直接挑战 GCC 的地位。2012年，LLVM 获得美国计算机学会 ACM 的软件系统大奖，和 UNIX，WWW，TCP/IP，Tex，JAVA 等齐名。
>  Chris Lattner 生于 1978 年，2005年加入苹果，将苹果使用的 GCC 全面转为 LLVM。2010年开始主导开发 Swift 语言。

LLVM是构架[编译器](https://links.jianshu.com/go?to=https%3A%2F%2Fbaike.baidu.com%2Fitem%2F%E7%BC%96%E8%AF%91%E5%99%A8%2F8853067)(compiler)的框架系统，以C++编写而成，用于优化以任意程序语言编写的程序的编译时间(compile-time)、链接时间(link-time)、运行时间(run-time)以及空闲时间(idle-time)，对开发者保持开放，并兼容已有脚本。

LLVM计划启动于2000年，最初由美国UIUC大学的Chris Lattner博士主持开展。2006年Chris Lattner加盟Apple Inc.并致力于LLVM在Apple开发体系中的应用。Apple也是LLVM计划的主要资助者。

目前LLVM已经被苹果IOS开发工具、Xilinx Vivado、Facebook、Google等各大公司采用。

LLVM是一个模块化和可重用的编译器和工具链技术的集合，Clang 是 LLVM 的子项目，是 C，C++ 和 Objective-C 编译器，目的是提供惊人的快速编译，比 GCC 快3倍，其中的 Clang Static Analyzer 主要是进行语法分析，语义分析和生成中间代码，当然这个过程会对代码进行检查，出错的和需要警告的会标注出来。LLVM 核心库提供一个优化器，对流行的 CPU 做代码生成支持。lld 是 Clang / LLVM 的内置链接器，clang 必须调用链接器来产生可执行文件。

> 引用：两篇入门文章 [**Blogs**](https://links.jianshu.com/go?to=https%3A%2F%2Fgithub.com%2FCYBoys%2FBlogs)  还有一个关于代码规范的插件
>  1、[LLVM & Clang 入门](https://links.jianshu.com/go?to=https%3A%2F%2Fgithub.com%2FCYBoys%2FBlogs%2Fblob%2Fmaster%2FLLVM_Clang%2FLLVM%20%26%20Clang%20%E5%85%A5%E9%97%A8.md)
>  2、[Clang Plugin 之 Debug](https://links.jianshu.com/go?to=https%3A%2F%2Fgithub.com%2FCYBoys%2FBlogs%2Fblob%2Fmaster%2FLLVM_Clang%2FClang%20Plugin%20%E4%B9%8B%20Debug.md)
>  其他
>  [iOS 性能优化总结](https://links.jianshu.com/go?to=https%3A%2F%2Fgithub.com%2FCYBoys%2FBlogs%2Fblob%2Fmaster%2FiOS%2Fios-performance-optimization-summary.md)
>  [iOS 持续交付之 Fastlane](https://links.jianshu.com/go?to=https%3A%2F%2Fgithub.com%2FCYBoys%2FBlogs%2Fblob%2Fmaster%2FiOS%2Fios-fastlane.md)





### **iOS编译**

> 在苹果公司使用的编译器是 LLVM，相比于 Xcode 5 版本前使用的 GCC，编译速度提高了 3 倍。同时，苹果公司也反过来主导了 LLVM 的发展，让 LLVM 可以针对苹果公司的硬件进行更多的优化。
>
>  总结来说，LLVM 是编译器工具链技术的一个集合。而其中的 lld 项目，就是内置链接器。编译器会对每个文件进行编译，生成 Mach-O（可执行文件）；链接器会将项目中的多个 Mach-O 文件合并成一个。
>
>  LLVM 的编译过程非常复杂。如果你有兴趣的话，可以通过[官方手册](https://links.jianshu.com/go?to=http%3A%2F%2Fllvm.org%2Fdocs%2F)查看完整的编译过程。

#### 总结下编译的几个主要过程：

- 首先，你写好代码后，LLVM 会预处理你的代码，比如把宏嵌入到对应的位置。
- 预处理完后，LLVM 会对代码进行词法分析和语法分析，生成 AST 。AST 是抽象语法树，结构上比代码更精简，遍历起来更快，所以使用 AST 能够更快速地进行静态检查，同时还能更快地生成 IR（中间表示）。
- 最后 AST 会生成 IR，IR 是一种更接近机器码的语言，区别在于和平台无关，通过 IR 可以生成多份适合不同平台的机器码。对于 iOS 系统，IR 生成的可执行文件就是 Mach-O。

编译器：每个文件进行编译 -> Mach-O（可执行文件）
 链接器：项目中的多个 Mach-O 文件 -> 合并成一个

编译的主要过程：
 Source Code --> AST -->  IR --> （机器码：指令集）armv7/64/x86 CUP的指令集（Mach-O文件）

源码、抽象语法树、中间表示语言





##### 引用：资源链接

- [Clang](https://links.jianshu.com/go?to=http%3A%2F%2Fclang.llvm.org%2F)
- [LLVM](https://links.jianshu.com/go?to=http%3A%2F%2Fllvm.org%2F)
- [iOS编译：从命令行运行分析器](https://links.jianshu.com/go?to=http%3A%2F%2Fclang-analyzer.llvm.org%2Fscan-build)
- [检查项](https://links.jianshu.com/go?to=http%3A%2F%2Fclang-analyzer.llvm.org%2Favailable_checks.html)
- [Clang常见的误报问题及处理方法](https://links.jianshu.com/go?to=http%3A%2F%2Fclang-analyzer.llvm.org%2Ffaq.html)
- Clang静态分析器：[Clang Static Analyzer](https://links.jianshu.com/go?to=http%3A%2F%2Fclang-analyzer.llvm.org%2F)
- [开源地址](https://links.jianshu.com/go?to=https%3A%2F%2Fgithub.com%2Fsearch%3Fp%3D1%26q%3DClang%2BStatic%2BAnalyzer%26type%3DRepositories%26utf8%3D%E2%9C%93)
- [GNU](https://links.jianshu.com/go?to=http%3A%2F%2Fgnustep.org%2F)

参考内容：
 [深入浅出iOS编译](https://links.jianshu.com/go?to=https%3A%2F%2Fgithub.com%2FLeoMobileDeveloper%2FBlogs%2Fblob%2Fmaster%2FCompiler%2Fxcode-compile-deep.md)
 [iOS编译过程的原理和应用](https://links.jianshu.com/go?to=https%3A%2F%2Fgithub.com%2FLeoMobileDeveloper%2FBlogs%2Fblob%2Fmaster%2FiOS%2FiOS%E7%BC%96%E8%AF%91%E8%BF%87%E7%A8%8B%E7%9A%84%E5%8E%9F%E7%90%86%E5%92%8C%E5%BA%94%E7%94%A8.md)
 [iOS汇编快速入门上篇](https://links.jianshu.com/go?to=https%3A%2F%2Fgithub.com%2FLeoMobileDeveloper%2FBlogs%2Fblob%2Fmaster%2FBasic%2FiOS%20assembly%20toturial%20part%201.md)



























# [iOS编译过程的原理和应用](https://github.com/LeoMobileDeveloper/Blogs/blob/master/iOS/iOS%E7%BC%96%E8%AF%91%E8%BF%87%E7%A8%8B%E7%9A%84%E5%8E%9F%E7%90%86%E5%92%8C%E5%BA%94%E7%94%A8.md#%E5%89%8D%E8%A8%80)



## 前言

一般可以将编程语言分为两种，[编译语言](https://zh.wikipedia.org/wiki/編譯語言)和[直译式语言](https://en.wikipedia.org/wiki/Interpreted_language)。

像C++,Objective C都是编译语言。编译语言在执行的时候，必须先通过编译器生成机器码，机器码可以直接在CPU上执行，所以执行效率较高。

像JavaScript,Python都是直译式语言。直译式语言不需要经过编译的过程，而是在执行的时候通过一个中间的解释器将代码解释为CPU可以执行的代码。所以，较编译语言来说，直译式语言效率低一些，但是编写的更灵活，也就是为啥JS大法好。

iOS开发目前的常用语言是：Objective和Swift。二者都是编译语言，换句话说都是需要编译才能执行的。二者的编译都是依赖于Clang + LLVM. 篇幅限制，本文只关注Objective C，因为原理上大同小异。

可能会有同学想问，我不懂编译的过程，写代码也没问题啊？这点我是不否定的。但是，充分理解了编译的过程，会对你的开发大有帮助。本文的最后，会以以下几个例子，来讲解如何合理利用XCode和编译

- `__attribute__`
- Clang警告处理
- 预处理
- 插入编译期脚本
- 提高项目编译速度

对于不想看我啰里八嗦讲一大堆原理的同学，可以直接跳到本文的最后一个章节。

------

## iOS编译

Objective C采用Clang(swift采用[swiftc](https://swift.org/compiler-stdlib/#compiler-architecture))作为编译器前端，LLVM作为编译器后端。

简单的编译过程如图

[![img](https://github.com/LeoMobileDeveloper/Blogs/raw/master/iOS/images/compile_1.png)](https://github.com/LeoMobileDeveloper/Blogs/blob/master/iOS/images/compile_1.png)

### 编译器前端

> 编译器前端的任务是进行：词法分析，语法分析，语义分析，生成中间代码(intermediate representation )。在这个过程中，会进行类型检查，如果发现错误或者警告会标注出来在哪一行。

[![img](https://github.com/LeoMobileDeveloper/Blogs/raw/master/iOS/images/compile_2.png)](https://github.com/LeoMobileDeveloper/Blogs/blob/master/iOS/images/compile_2.png)

### 编译器后端

> 编译器后端会进行机器无关的代码优化，生成机器语言，并且进行机器相关的代码优化。iOS的编译过程，后端的处理如下

- **LVVM优化器会进行BitCode的生成，链接期优化等等**。

[![img](https://github.com/LeoMobileDeveloper/Blogs/raw/master/iOS/images/compile_3.png)](https://github.com/LeoMobileDeveloper/Blogs/blob/master/iOS/images/compile_3.png)

- **LLVM机器码生成器会针对不同的架构，比如arm64等生成不同的机器码**。

[![img](https://github.com/LeoMobileDeveloper/Blogs/raw/master/iOS/images/compile_4.png)](https://github.com/LeoMobileDeveloper/Blogs/blob/master/iOS/images/compile_4.png)

------

## 执行一次XCode build的流程

当你在XCode中，选择build的时候（快捷键command+B），会执行如下过程

- 编译信息写入辅助文件，创建编译后的文件架构(name.app)
- 处理文件打包信息，例如在debug环境下

```
Entitlements:
{
    "application-identifier" = "app的bundleid";
    "aps-environment" = development;
}
```

- 执行CocoaPod编译前脚本
  - 例如对于使用CocoaPod的工程会执行`CheckPods Manifest.lock`
- 编译各个.m文件，使用`CompileC`和`clang`命令。

```
CompileC ClassName.o ClassName.m normal x86_64 objective-c com.apple.compilers.llvm.clang.1_0.compiler
export LANG=en_US.US-ASCII
export PATH="..."
clang -x objective-c -arch x86_64 -fmessage-length=0 -fobjc-arc... -Wno-missing-field-initializers ... -DDEBUG=1 ... -isysroot iPhoneSimulator10.1.sdk -fasm-blocks ... -I 上文提到的文件 -F 所需要的Framework  -iquote 所需要的Framework  ... -c ClassName.c -o ClassName.o
```

通过这个编译的命令，我们可以看到

```
clang是实际的编译命令
-x 		objective-c 指定了编译的语言
-arch 	x86_64制定了编译的架构，类似还有arm7等
-fobjc-arc 一些列-f开头的，指定了采用arc等信息。这个也就是为什么你可以对单独的一个.m文件采用非ARC编程。
-Wno-missing-field-initializers 一系列以-W开头的，指的是编译的警告选项，通过这些你可以定制化编译选项
-DDEBUG=1 一些列-D开头的，指的是预编译宏，通过这些宏可以实现条件编译
-iPhoneSimulator10.1.sdk 制定了编译采用的iOS SDK版本
-I 把编译信息写入指定的辅助文件
-F 链接所需要的Framework
-c ClassName.c 编译文件
-o ClassName.o 编译产物
```

- 链接需要的Framework，例如`Foundation.framework`,`AFNetworking.framework`,`ALiPay.fframework`
- 编译xib文件
- 拷贝xib，图片等资源文件到结果目录
- 编译ImageAssets
- 处理info.plist
- 执行CocoaPod脚本
- 拷贝Swift标准库
- 创建.app文件和对其签名

------

## IPA包的内容

例如，我们通过iTunes Store下载微信，然后获得ipa安装包，然后实际看看其安装包的内容。

[![img](https://github.com/LeoMobileDeveloper/Blogs/raw/master/iOS/images/compile_5.png)](https://github.com/LeoMobileDeveloper/Blogs/blob/master/iOS/images/compile_5.png)

- 右键ipa，重命名为`.zip`
- 双击zip文件，解压缩后会得到一个文件夹。所以，ipa包就是一个普通的压缩包。

[![img](https://github.com/LeoMobileDeveloper/Blogs/raw/master/iOS/images/compile_6.png)](https://github.com/LeoMobileDeveloper/Blogs/blob/master/iOS/images/compile_6.png)

\- 右键图中的`WeChat`，选择显示包内容，然后就能够看到实际的ipa包内容了。

------

## 二进制文件的内容

通过XCode的Link Map File，我们可以窥探二进制文件中布局。 在XCode -> Build Settings -> 搜索map -> 开启Write Link Map File

[![img](https://github.com/LeoMobileDeveloper/Blogs/raw/master/iOS/images/compile_7.png)](https://github.com/LeoMobileDeveloper/Blogs/blob/master/iOS/images/compile_7.png)

开启后，在编译，我们可以在对应的Debug/Release目录下看到对应的link map的text文件。 默认的目录在

```
~/Library/Developer/Xcode/DerivedData/<TARGET-NAME>-对应ID/Build/Intermediates/<TARGET-NAME>.build/Debug-iphoneos/<TARGET-NAME>.build/
```

例如，我的TargetName是`EPlusPan4Phone`，目录如下

```
/Users/huangwenchen/Library/Developer/Xcode/DerivedData/EPlusPan4Phone-eznmxzawtlhpmadnbyhafnpqpizo/Build/Intermediates/EPlusPan4Phone.build/Debug-iphonesimulator/EPlusPan4Phone.build
```

这个映射文件的主要包含以下部分：

### **Object files**

这个部分包括的内容

- .o 文文件，也就是上文提到的.m文件编译后的结果。
- .a文件
- 需要link的framework

> \#！ Arch: x86_64 #Object files: [0] linker synthesized [1] /EPlusPan4Phone.build/EPlusPan4Phone.app.xcent [2]/EPlusPan4Phone.build/Objects-normal/x86_64/ULWBigResponseButton.o ... [1175]/UMSocial_Sdk_4.4/libUMSocial_Sdk_4.4.a(UMSocialJob.o) [1188]/iPhoneSimulator10.1.sdk/System/Library/Frameworks//Foundation.framework/Foundation

这个区域的存储内容比较简单：前面是文件的编号，后面是文件的路径。文件的编号在后续会用到

## **Sections**

这个区域提供了各个段（Segment）和节（Section）在可执行文件中的位置和大小。这个区域完整的描述克可执行文件中的全部内容。

其中，段分为两种

- __TEXT 代码段
- __DATA 数据段

例如，之前写的一个App，Sections区域如下，可以看到，代码段的

__text节的地址是0x1000021B0，大小是0x0077EBC3，而二者相加的下一个位置正好是__stubs的位置0x100780D74。

```
# Sections:
# 位置       大小        段       节
# Address	Size    	Segment	Section
0x1000021B0	0x0077EBC3	__TEXT	__text //代码
0x100780D74	0x00000FD8	__TEXT	__stubs
0x100781D4C	0x00001A50	__TEXT	__stub_helper
0x1007837A0	0x0001AD78	__TEXT	__const //常量
0x10079E518	0x00041EF7	__TEXT	__objc_methname //OC 方法名
0x1007E040F	0x00006E34	__TEXT	__objc_classname //OC 类名
0x1007E7243	0x00010498	__TEXT	__objc_methtype  //OC 方法类型
0x1007F76DC	0x0000E760	__TEXT	__gcc_except_tab 
0x100805E40	0x00071693	__TEXT	__cstring  //字符串
0x1008774D4	0x00004A9A	__TEXT	__ustring  
0x10087BF6E	0x00000149	__TEXT	__entitlements 
0x10087C0B8	0x0000D56C	__TEXT	__unwind_info 
0x100889628	0x000129C0	__TEXT	__eh_frame
0x10089C000	0x00000010	__DATA	__nl_symbol_ptr
0x10089C010	0x000012C8	__DATA	__got
0x10089D2D8	0x00001520	__DATA	__la_symbol_ptr
0x10089E7F8	0x00000038	__DATA	__mod_init_func
0x10089E840	0x0003E140	__DATA	__const //常量
0x1008DC980	0x0002D840	__DATA	__cfstring
0x10090A1C0	0x000022D8	__DATA	__objc_classlist // OC 方法列表
0x10090C498	0x00000010	__DATA	__objc_nlclslist 
0x10090C4A8	0x00000218	__DATA	__objc_catlist
0x10090C6C0	0x00000008	__DATA	__objc_nlcatlist
0x10090C6C8	0x00000510	__DATA	__objc_protolist // OC协议列表
0x10090CBD8	0x00000008	__DATA	__objc_imageinfo
0x10090CBE0	0x00129280	__DATA	__objc_const // OC 常量
0x100A35E60	0x00010908	__DATA	__objc_selrefs
0x100A46768	0x00000038	__DATA	__objc_protorefs 
0x100A467A0	0x000020E8	__DATA	__objc_classrefs 
0x100A48888	0x000019C0	__DATA	__objc_superrefs // OC 父类引用
0x100A4A248	0x0000A500	__DATA	__objc_ivar // OC ivar
0x100A54748	0x00015CC0	__DATA	__objc_data
0x100A6A420	0x00007A30	__DATA	__data
0x100A71E60	0x0005AF70	__DATA	__bss
0x100ACCDE0	0x00053A4C	__DATA	__common
```

## **Symbols**

Section部分将二进制文件进行了一级划分。而，Symbols对Section中的各个段进行了二级划分， 例如，对于`__TEXT __text`,表示代码段中的代码内容。

```
0x1000021B0	0x0077EBC3	__TEXT	__text //代码
```

而对应的`Symbols`，起始地址也是`0x1000021B0 `。其中，文件编号和上文的编号对应

```
[2]/EPlusPan4Phone.build/Objects-normal/x86_64/ULWBigResponseButton.o
```

具体内容如下

```
# Symbols:
  地址     大小          文件编号    方法名
# Address	Size    	File       Name
0x1000021B0	0x00000109	[  2]     -[ULWBigResponseButton pointInside:withEvent:]
0x1000022C0	0x00000080	[  3]     -[ULWCategoryController liveAPI]
0x100002340	0x00000080	[  3]     -[ULWCategoryController categories]
....
```

到这里，我们知道OC的方法是如何存储的，我们再来看看ivar是如何存储的。 首先找到数据栈中`__DATA __objc_ivar`

```
0x100A4A248	0x0000A500	__DATA	__objc_ivar
```

然后，搜索这个地址`0x100A4A248`，就能找到ivar的存储区域。

```
0x100A4A248	0x00000008	[  3] _OBJC_IVAR_$_ULWCategoryController._liveAPI
```

值得一提的是，对于String，会显式的存储到数据段中，例如,

```
0x1008065C2	0x00000029	[ 11] literal string: http://sns.whalecloud.com/sina2/callback
```

> 所以，若果你的加密Key以明文的形式写在文件里，是一件很危险的事情。

------

## dSYM 文件

我们在每次编译过后，都会生成一个dsym文件。dsym文件中，存储了16进制的函数地址映射。

在App实际执行的二进制文件中，是通过地址来调用方法的。在App crash的时候，第三方工具（Fabric,友盟等）会帮我们抓到崩溃的调用栈，调用栈里会包含crash地址的调用信息。然后，通过dSYM文件，我们就可以由地址映射到具体的函数位置。

XCode中，选择Window -> Organizer可以看到我们生成的archier文件

[![img](https://github.com/LeoMobileDeveloper/Blogs/raw/master/iOS/images/compile_8.png)](https://github.com/LeoMobileDeveloper/Blogs/blob/master/iOS/images/compile_8.png)

然后，

- 右键 -> 在finder中显示。
- 右键 -> 查看包内容。

关于如何用dsym文件来分析崩溃位置，可以查看我之前的一篇博客。

- [iOS 如何调试第三方统计到的崩溃报告](http://blog.csdn.net/hello_hwc/article/details/50036323)

------

## 那些你想到和想不到的应用场景

### `__attribute__`

或多或少，你都会在第三方库或者iOS的头文件中，见到过__attribute__。 比如

```
__attribute__ ((warn_unused_result)) //如果没有使用返回值，编译的时候给出警告
```

> `__attribtue__` 是一个高级的的编译器指令，它允许开发者指定更更多的编译检查和一些高级的编译期优化。

分为三种：

> - 函数属性 （Function Attribute）
> - 类型属性 (Variable Attribute )
> - 变量属性 (Type Attribute )

语法结构

`__attribute__` 语法格式为：`__attribute__ ((attribute-list))` 放在声明分号“;”前面。

比如，在三方库中最常见的，声明一个属性或者方法在当前版本弃用了

```
@property (strong,nonatomic)CLASSNAME * property __deprecated;
```

这样的好处是：给开发者一个过渡的版本，让开发者知道这个属性被弃用了，应当使用最新的API，但是被__deprecated的属性仍然可以正常使用。如果直接弃用，会导致开发者在更新Pod的时候，代码无法运行了。

`__attribtue__`的使用场景很多，本文只列举iOS开发中常用的几个：

```
//弃用API，用作API更新
#define __deprecated	__attribute__((deprecated)) 

//带描述信息的弃用
#define __deprecated_msg(_msg) __attribute__((deprecated(_msg)))

//遇到__unavailable的变量/方法，编译器直接抛出Error
#define __unavailable	__attribute__((unavailable))

//告诉编译器，即使这个变量/方法 没被使用，也不要抛出警告
#define __unused	__attribute__((unused))

//和__unused相反
#define __used		__attribute__((used))

//如果不使用方法的返回值，进行警告
#define __result_use_check __attribute__((__warn_unused_result__))

//OC方法在Swift中不可用
#define __swift_unavailable(_msg)	__attribute__((__availability__(swift, unavailable, message=_msg)))
```

### Clang警告处理

你一定还见过如下代码：

```
#pragma clang diagnostic push
#pragma clang diagnostic ignored "-Wundeclared-selector"
///代码
#pragma clang diagnostic pop
```

这段代码的作用是

1. 对当前编译环境进行压栈
2. 忽略`-Wundeclared-selector`（未声明的）Selector警告
3. 编译代码
4. 对编译环境进行出栈

通过clang diagnostic push/pop,你可以灵活的控制代码块的编译选项。

我在之前的一篇文章里，详细的介绍了XCode的警告相关内容。本文篇幅限制，就不详细讲解了。

- [iOS 合理利用Clang警告来提高代码质量](http://blog.csdn.net/Hello_Hwc/article/details/46425503)

在这个链接，你可以找到所有的Clang warnings警告

- [fuckingclangwarnings](http://fuckingclangwarnings.com/)

### 预处理

所谓预处理，就是在编译之前的处理。预处理能够让你定义编译器变量，实现条件编译。 比如，这样的代码很常见

```
#ifdef DEBUG
//...
#else
//...
#endif
```

同样，我们同样也可以定义其他预处理变量,在XCode-选中Target-build settings中，搜索preprocess。然后点击图中蓝色的加号，可以分别为debug和release两种模式设置预处理宏。 比如我们加上：`TestServer`，表示在这个宏中的代码运行在测试服务器

[![img](https://github.com/LeoMobileDeveloper/Blogs/raw/master/iOS/images/compile_9.png)](https://github.com/LeoMobileDeveloper/Blogs/blob/master/iOS/images/compile_9.png)

然后，配合多个Target（右键Target，选择Duplicate），单独一个Target负责测试服务器。这样我们就不用每次切换测试服务器都要修改代码了。

```
#ifdef TESTMODE
//测试服务器相关的代码
#else
//生产服务器相关代码
#endif
```

### 插入脚本

通常，如果你使用CocoaPod来管理三方库，那么你的Build Phase是这样子的：

[![img](https://github.com/LeoMobileDeveloper/Blogs/raw/master/iOS/images/compile_10.png)](https://github.com/LeoMobileDeveloper/Blogs/blob/master/iOS/images/compile_10.png)

其中：[CP]开头的，就是CocoaPod插入的脚本。

- Check Pods Manifest.lock，用来检查cocoapod管理的三方库是否需要更新
- Embed Pods Framework，运行脚本来链接三方库的静态/动态库
- Copy Pods Resources，运行脚本来拷贝三方库的资源文件

而这些配置信息都存储在这个文件(.xcodeproj)里

[![img](https://github.com/LeoMobileDeveloper/Blogs/raw/master/iOS/images/compile_11.png)](https://github.com/LeoMobileDeveloper/Blogs/blob/master/iOS/images/compile_11.png)

到这里，CocoaPod的原理也就大致搞清楚了，通过修改xcodeproject，然后配置编译期脚本，来保证三方库能够正确的编译连接。

同样，我们也可以插入自己的脚本，来做一些额外的事情。比如，每次进行archive的时候，我们都必须手动调整target的build版本，如果一不小心，就会忘记。这个过程，我们可以通过插入脚本自动化。

```
buildNumber=$(/usr/libexec/PlistBuddy -c "Print CFBundleVersion" "${PROJECT_DIR}/${INFOPLIST_FILE}")
buildNumber=$(($buildNumber + 1))
/usr/libexec/PlistBuddy -c "Set :CFBundleVersion $buildNumber" "${PROJECT_DIR}/${INFOPLIST_FILE}"
```

这段脚本其实很简单，读取当前pist的build版本号,然后对其加一，重新写入。

使用起来也很简单：

- Xcode - 选中Target - 选中build phase
- 选择添加Run Script Phase

[![img](https://github.com/LeoMobileDeveloper/Blogs/raw/master/iOS/images/compile_12.png)](https://github.com/LeoMobileDeveloper/Blogs/blob/master/iOS/images/compile_12.png)

- 然后把这段脚本拷贝进去，并且勾选Run Script Only When installing，保证只有Archive时才会执行这段脚本。重命名脚本的名字为Auto Increase build number
- 然后，拖动这个脚本的到Link Binary With Libraries下面

[![img](https://github.com/LeoMobileDeveloper/Blogs/raw/master/iOS/images/compile_14.png)](https://github.com/LeoMobileDeveloper/Blogs/blob/master/iOS/images/compile_14.png)

### 脚本编译打包

脚本化编译打包对于CI（持续集成）来说，十分有用。iOS开发中，编译打包必备的两个命令是：

```
//编译成.app
xcodebuild  -workspace $projectName.xcworkspace -scheme $projectName  -configuration $buildConfig clean build SYMROOT=$buildAppToDir
//打包
xcrun -sdk iphoneos PackageApplication -v $appDir/$projectName.app -o $appDir/$ipaName.ipa

通过info命令，可以查看到详细的文档
info xcodebuild
```

- [完整的脚本](https://github.com/LeoMobileDeveloper/Blogs/blob/master/DemoProjects/Scripts/autoIPA.sh)，使用的时候，需要拷贝到工程的根目录

### 提高项目编译速度

通常，当项目很大，源代码和三方库引入很多的时候，我们会发现编译的速度很慢。在了解了XCode的编译过程后，我们可以从以下角度来优化编译速度：

#### 查看编译时间

我们需要一个途径，能够看到编译的时间，这样才能有个对比，知道我们的优化究竟有没有效果。 对于XCode 8，关闭XCode，终端输入以下指令

```
$ defaults write com.apple.dt.Xcode ShowBuildOperationDuration YES
```

然后，重启XCode，然后编译，你会在这里看到编译时间。

[![img](https://github.com/LeoMobileDeveloper/Blogs/raw/master/iOS/images/compile_15.png)](https://github.com/LeoMobileDeveloper/Blogs/blob/master/iOS/images/compile_15.png)

代码层面的优化

#### **forward declaration**

所谓`forward declaration`，就是`@class CLASSNAME`，而不是`#import CLASSNAME.h`。这样，编译器能大大提高#import的替换速度。

#### 对常用的工具类进行打包（Framework/.a）

打包成Framework或者静态库，这样编译的时候这部分代码就不需要重新编译了。

#### 常用头文件放到预编译文件里

XCode的pch文件是预编译文件，这里的内容在执行XCode build之前就已经被预编译，并且引入到每一个.m文件里了。

编译器选项优化

#### Debug模式下，不生成dsym文件

上文提到了，dysm文件里存储了调试信息，在Debug模式下，我们可以借助XCode和LLDB进行调试。所以，不需要生成额外的dsym文件来降低编译速度。

#### Debug开启`Build Active Architecture Only`

在XCode -> Build Settings -> Build Active Architecture Only 改为YES。这样做，可以只编译当前的版本，比如arm7/arm64等等，记得只开启Debug模式。这个选项在高版本的XCode中自动开启了。

#### Debug模式下，关闭编译器优化

编译器优化

[![img](https://github.com/LeoMobileDeveloper/Blogs/raw/master/iOS/images/compile_16.png)](https://github.com/LeoMobileDeveloper/Blogs/blob/master/iOS/images/compile_16.png)

