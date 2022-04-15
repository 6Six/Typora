# [iOS编译原理与应用](https://juejin.cn/post/6844903896230395918)



# 引言

在Xcode中，当我们按下command + B进行build操作后发生了那些事情，这是一个将代码编译的过程。Xcode现在使用的编译器是LLVM，Xcode 早期使用的是GCC编译器，由于一些历史原因，从Xcode5开始正式过渡到使用LLVM编译器。下文将着重介绍LLVM。

# 编译原理

## LLVM简介



![img](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2019/7/24/16c22fea55748d58~tplv-t2oaga2asx-zoom-in-crop-mark:1304:0:0:0.awebp)



- LLVM项目是模块化、可重用的编译器以及工具链技术的集合。
- 美国计算机协会（ACM）将其 2012 年软件系统奖颁给了LLVM，之前曾获得此奖项的软件和技术包括：Java、Apache、Mosaic、the World Wide Web、SmallTalk、UNIX、Eclipse等等。
- LLVM项目的发展起源于2000年伊利诺伊大学厄巴纳-香槟分校维克拉姆·艾夫（Vikram Adve）与克里斯·拉特纳（Chris Lattner）的研究，他们想要为所有静态及动态语言创造出动态的编译技术。LLVM是以BSD授权来发展的开源软件。2005年，苹果电脑雇用了克里斯·拉特纳及他的团队为苹果电脑开发应用程序系统，LLVM为现今Mac OS X及iOS开发工具的一部分。
- LLVM的命名最早源自于底层虚拟机（Low Level Virtual Machine）的首字母缩写，由于这个项目的范围并不局限于创建一个虚拟机，这个缩写导致了广泛的疑惑。官方描述如下：The name “LLVM” itself is not an acronym；it is the full name of the project。LLVM这个名称并不是首字母缩略词，它是项目的全名。
- LLVM开始成长之后，成为众多编译工具及低级工具技术的统称，使得这个名字变得更不贴切，开发者因而决定放弃这个缩写的意涵，现今LLVM已单纯成为一个品牌，适用于LLVM下的所有项目，包含LLVM中介码（LLVM IR）、LLVM除错工具、LLVM C++标准库等。
- 目前NDK/Xcode均采用LLVM作为默认的编译器。

## 传统的编译器架构



![img](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2019/7/24/16c2301ada5faae3~tplv-t2oaga2asx-zoom-in-crop-mark:1304:0:0:0.awebp)



- Frontend：前端，对源码做词法分析、语法分析、语义分析、生成中间代码
- Optimizer：优化器，用于中间代码优化
- Backend：后端，用于生成机器码

## LLVM架构



![img](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2019/7/24/16c230372fc40295~tplv-t2oaga2asx-zoom-in-crop-mark:1304:0:0:0.awebp)



- 前端将各种类型的源代码编译为中间代码，也就是bitcode，在LLVM体系内，不同的语言有不同的编译器前端，常见的如clang负责 c/c++/oc的编译，flang负责fortran的编译，swiftc负责swift的编译等等。
- 不同的前后端使用统一的中间代码LLVM Intermediate Representation(LLVM IR)。
- 优化阶段是一个通用的阶段，针对的是统一的LLVM IR，无论是新的编程语言，还是支持新的硬件设备，都不需要对优化阶段做修改，具体是对bitcode进行各种类型的优化，将bitcode代码进行一些逻辑的转换，使得代码效率更高，体积更小，比如DeadStrip/SimplifyCFG。
- 后端，也叫CodeGenerator，负责把优化后的bitcode编译为指定目标架构的机器码，比如 X86Backend负责把bitcode编译为x86指令集的机器码。
- GCC相比之下，前后端耦合在了一起。所以，GCC支持一门新的语言，或是为了支持一个新的平台，就变得异常困难。
- LLVM现在被作为实现各种静态和运行时编译语言通用基础架构（GCC 家族、Java、.Net、Python、Ruby、Scheme、Haskell、D等）。
- LLVM体系中，不同语言源代码将会被转化为统一的bitcode格式，三个模块相互独立，可以充分复用。比如，如果开发一门新的语言，只要制造一个该语言的前端，将源码编译为bitcode，优化和后端不用管。同理，如果新的芯片架构问世，只需基于LLVM重新编写一套目标平台的后端即可。

## Clang

- LLVM项目的一个子项目。
- 基于LLVM架构的C/C++/Objective-C/Objective-C++编译器前端。
- 相比于 GCC，Clang具有如下优点：
- 编译速度快：在某些平台上，Clang的编译速度显著的快过GCC（Debug 模式下编译 OC 速度比 GCC 快 3 倍）；
- 占用内存小：Clang生成的AST所占用的内存是GCC的五分之一左右；
- 模块化设计：Clang采用基于库的模块化设计，易于IDE集成及其他用途的重用；
- 诊断信息可读性强：在编译过程中，Clang创建并保留了大量详细的元数据（metadata），有利于调试和错误报告；
- 设计清晰简单，容易理解，易于扩展增强。

客观的说GCC也有很多优点：例如支持多平台，基于C无需 C++编译器即可编译。这个优点到苹果那里反而成了缺点，苹果需要的是快。

## Clang与LLVM



![img](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2019/7/24/16c2308b0b439c74~tplv-t2oaga2asx-zoom-in-crop-mark:1304:0:0:0.awebp)



- 广义的LLVM：整个LLVM架构
- 狭义的LLVM：LLVM后端（代码优化、目标代码生成等）



![img](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2019/7/24/16c2308efb88f492~tplv-t2oaga2asx-zoom-in-crop-mark:1304:0:0:0.awebp)



## OC源文件的编译过程

- 命令行查看编译的过程：

```
clang -ccc-print-phases main.m
复制代码
```



![img](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2019/7/24/16c2309eafbce5c5~tplv-t2oaga2asx-zoom-in-crop-mark:1304:0:0:0.awebp)



- 查看preprocessor（预处理）的结果：

```
clang -E main.m -F /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneSimulator.platform/Developer/SDKs/iPhoneSimulator.sdk/System/Library/Frameworks 
复制代码
```



![img](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2019/7/24/16c230a84fef3488~tplv-t2oaga2asx-zoom-in-crop-mark:1304:0:0:0.awebp)



### 1. 词法分析

词法分析，生成Token

```
int sum(int a,int b){
    int c = a + b;
    return c;
}
复制代码
clang -fmodules -E -Xclang -dump-tokens main.m
复制代码
```



![img](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2019/7/24/16c230bcff1b9eb8~tplv-t2oaga2asx-zoom-in-crop-mark:1304:0:0:0.awebp)



这个命令的作用是，显示每个Token的类型、值，以及位置。参考该链接，可以看到Clang定义的所有Token类型。 可以分为下面这4类：

- 关键字：语法中的关键字，比如 if、else、while、for 等；
- 标识符：变量名；
- 字面量：值、数字、字符串；
- 特殊符号：加减乘除等符号。

### 2. 语法分析

利用上面输出的Token先按照语法组合成语义，生成类似VarDecl这样的节点，然后将这些节点按照层级关系构成抽象语法树（AST）。

- 语法分析，生成语法树（AST，Abstract Syntax Tree）

```
clang -fmodules -fsyntax-only -Xclang -ast-dump main.m
复制代码
```

TranslationUnitDecl是根节点，表示一个编译单元；Decl表示一个声明；Expr表示的是表达式；Literal表示字面量，是一个特殊的Expr；Stmt表示陈述。

除此之外，Clang还有众多种类的节点类型。Clang里，节点主要分成Type类型、Decl声明、Stmt陈述这三种，其他的都是这三种的派生。通过扩展这三类节点，就能够将无限的代码形态用有限的形式来表现出来。



![img](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2019/7/24/16c230e733a442bc~tplv-t2oaga2asx-zoom-in-crop-mark:1304:0:0:0.awebp)





![img](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2019/7/24/16c230ee06a7b486~tplv-t2oaga2asx-zoom-in-crop-mark:1304:0:0:0.awebp)



### 3. LLVM IR

LLVM IR有3种表示形式，但本质上是等价的。

- text：便于阅读的文本格式，类似于汇编语言，拓展名 .ll

```
clang -S -emit-llvm main.m
复制代码
```

- memory：内存格式
- bitcode：二进制格式，拓展名 .bc

```
clang -c -emit-llvm main.m
复制代码
```

### 4. IR基本语法

.ll 文件部分内容，如下：



![img](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2019/7/24/16c2310ae9387652~tplv-t2oaga2asx-zoom-in-crop-mark:1304:0:0:0.awebp)



- 注释以分号 ; 开头
- 全局标识符以@开头，局部标示符以%开头
- alloca，在当前函数栈帧中分配内存
- i32 ，32bit，4 个字节
- align，内存对齐
- store，写入数据
- load，读取数据

# 应用与实践

基于LLVM 、Clang可以做很多实践，如下：

- LibClang、LibTooling、Clang Plugin

官方参考：

[clang.llvm.org/docs/Toolin…](https://link.juejin.cn?target=https%3A%2F%2Fclang.llvm.org%2Fdocs%2FTooling.html)

应用：语法树分析、语言转换等

- OCLint、Clang静态分析器（Clang Static Analyzer）
- Clang插件开发

官方参考：

[clang.llvm.org/docs/ClangP…](https://link.juejin.cn?target=https%3A%2F%2Fclang.llvm.org%2Fdocs%2FClangPlugins.html)

[clang.llvm.org/docs/Extern…](https://link.juejin.cn?target=https%3A%2F%2Fclang.llvm.org%2Fdocs%2FExternalClangExamples.html)

[clang.llvm.org/docs/RAVFro…](https://link.juejin.cn?target=https%3A%2F%2Fclang.llvm.org%2Fdocs%2FRAVFrontendAction.html)

应用：代码检查（命名规范、代码规范）等

- Pass开发

官方参考：

[llvm.org/docs/Writin…](https://link.juejin.cn?target=https%3A%2F%2Fllvm.org%2Fdocs%2FWritingAnLLVMPass.html)

应用：中间代码优化、代码混淆等

- 开发新的编程语言

[llvm-tutorial-cn.readthedocs.io/en/latest/i…](https://link.juejin.cn?target=https%3A%2F%2Fllvm-tutorial-cn.readthedocs.io%2Fen%2Flatest%2Findex.html)

[kaleidoscope-llvm-tutorial-zh-cn.readthedocs.io/zh_CN/lates…](https://link.juejin.cn?target=https%3A%2F%2Fkaleidoscope-llvm-tutorial-zh-cn.readthedocs.io%2Fzh_CN%2Flatest%2F)

## 编写及运行Clang插件

Clang使用模块化设计，可以将自身功能以库的方式来供上层应用来调用。比如，编码规范检查、IDE 中的语法高亮、语法检查等上层应用，都是使用Clang库的接口开发出来的。Clang有三个接口库可以供上层应用调用，分别是LibClang、Clang Plugin、LibTooling。

LibClang为了兼容更多Clang版本，相比Clang少了很多功能；Clang Plugin和LibTooling具备Clang 的全量能力。Clang Plugin编写代码的方式，和LibTooling几乎一样，不同的是Clang Plugin还能够控制编译过程，可以加warning或者直接中断编译提示错误。另外，编写好的LibTooling能够非常方便地转成Clang Plugin。 因此，Clang Plugin在功能上是最全的。

### 1. 源码下载

下载 LLVM Project

```
 git clone https://github.com/llvm/llvm-project.git
复制代码
```



![img](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2019/7/24/16c2316ccda46f08~tplv-t2oaga2asx-zoom-in-crop-mark:1304:0:0:0.awebp)



上图中，clang目录就是类C语言编译器的代码目录；llvm目录的代码包含两部分，一部分是对源码进行平台 无关优化的优化器代码，另一部分是生成平台相关汇编代码的生成器代码；lldb目录里是调试器的代码；lld里是链接器代码。

### 2. 源码编译

macOS属于类UNIX平台，因此既可以生成Makefile文件来编译，也可以生成Xcode工程来编译。

进入llvm-project文件目录，生成Makefile文件：

- 在llvm同级目录下新建一个llvm_make目录
- 在llvm_make中利用CMake进行编译

```
cmake -DLLVM_ENABLE_PROJECTS=clang -G "Unix Makefiles" ../llvm
复制代码
```

生成Xcode工程，可以使用如下命令

- 安装CMake工具
- 在llvm同级目录下新建一个llvm_xcode目录
- 在llvm_xcode中利用CMake进行编译

```
cmake -G Xcode -DLLVM_ENABLE_PROJECTS=clang ../llvm
复制代码
```

想要更多地了解CMake的语法和功能，你可以查看官方文档。

执行cmake命令时，你可能会遇到下面的提示：

```
-- The C compiler identification is unknown -- The CXX compiler identification is unknown CMake Error at CMakeLists.txt:39 (project):

No CMAKE_C_COMPILER could be found. 

CMake Error at CMakeLists.txt:39 (project):

No CMAKE_CXX_COMPILER could be found.
复制代码
```

这表明cmake没有找到代码编译器的命令行工具。分两种情况处理：

- 如果没有安装Xcode Commandline Tools的话，可以执行如下命令安装：

```
xcode-select --install
复制代码
```

- 如果你已经安装了Xcode Commandline Tools的话，直接reset即可

```
sudo xcode-select --reset
复制代码
```

生成Xcode工程后，打开生成的LLVM.xcodeproj文件，选择Automatically Create Schemes。

生成Xcode项目后再利用Xcode进行编译，但是速度很慢



![img](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2019/7/24/16c231a211d11607~tplv-t2oaga2asx-zoom-in-crop-mark:1304:0:0:0.awebp)



### 3. 插件目录

- 在clang/tools源码目录下新建一个插件目录，比如叫做mskj_plugin，添加MSKJPlugin.cpp文件和 CMakeLists.txt文件。其中，CMake编译需要通过CMakeLists.txt文件来指导编译，cpp是源文件。



![img](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2019/7/24/16c231ae4015671f~tplv-t2oaga2asx-zoom-in-crop-mark:1304:0:0:0.awebp)



- 在clang/tools目录下的CMakeList.txt文件当中最后一行加入：

```
add_clang_subdirectory(mskj-plugin)
复制代码
```

- 使用如下代码编写clang/tools/mskj-plugin/CMakeLists.txt文件，来定制编译流程：

```
add_llvm_library(MSKJPlugin MODULE MSKJPlugin.cpp PLUGIN_TOOL clang)
复制代码
```

MSKJPlugin是插件名，MSKJPlugin.cpp是源代码文件，这段代码是指，要将Clang插件代码集成到LLVM的Xcode工程中，并作为一个模块进行编写调试。添加了Clang插件的目录和文件后，再次用cmake命令生成Xcode工程，里面就能够集成MSKJPlugin.cpp文件。

### 4. 编写插件源码

① 编写PluginASTAction代码

由于Clang插件是没有main函数的，入口是PluginASTAction的ParseArgs函数。所以，编写Clang插件还要实现ParseArgs来处理入口参数。代码如下所示：

```
 class MSKJASTAction: public PluginASTAction {
    public:
        unique_ptr<ASTConsumer> CreateASTConsumer(CompilerInstance &ci, StringRef iFile) {
            return unique_ptr<MSKJASTConsumer> (new MSKJASTConsumer(ci));
        }
        
        bool ParseArgs(const CompilerInstance &ci, const vector<string> &args) {
            return true;
        }
    };
复制代码
```

② 编写ASTConsumer

FrontActions是编写Clang插件的入口，也是一个接口，是基于ASTFrontendAction的抽象基类。FrontActions为接下来基于AST操作的函数提供了一个入口和工作环境。

通过这个接口，你可以编写在编译过程中自定义的操作，具体方式是：通过ASTFrontendAction在 AST上自定义操作，重载CreateASTConsumer函数返回你自己的Consumer，以获取AST上的 ASTConsumer单元。ASTConsumer可以提供很多入口，是一个可以访问AST的抽象基类，可以重载 HandleTopLevelDecl()和 HandleTranslationUnit()两个函数，以接收访问AST时的回调。其中，HandleTopLevelDecl()函数是在访问到全局变量、函数定义这样最上层声明时进行回调，HandleTranslationUnit()函数会在接收每个节点访问时的回调。

```
class MSKJASTConsumer: public ASTConsumer {
    private:
        MatchFinder matcher;
        MSKJHandler handler;
        
    public:
        MSKJASTConsumer(CompilerInstance &ci) :handler(ci) {
            matcher.addMatcher(objcInterfaceDecl().bind("ObjCInterfaceDecl"), &handler);
        }
        
        void HandleTranslationUnit(ASTContext &context) {
            matcher.matchAST(context);
        }    
};
复制代码
```

③ 处理节点

```
 class MSKJHandler : public MatchFinder::MatchCallback {
    private:
        CompilerInstance &ci;
     
    public:
        MSKJHandler(CompilerInstance &ci) :ci(ci) {}
        
        void run(const MatchFinder::MatchResult &Result) {
            if (const ObjCInterfaceDecl *decl = Result.Nodes.getNodeAs<ObjCInterfaceDecl>("ObjCInterfaceDecl")) {
                size_t pos = decl->getName().find('_');
                if (pos != StringRef::npos) {
                    DiagnosticsEngine &D = ci.getDiagnostics();
                    SourceLocation loc = decl->getLocation().getLocWithOffset(pos);
                    D.Report(loc, D.getCustomDiagID(DiagnosticsEngine::Error, "MSKJ：类名中不能带有下划线"));
                }
            }
        }
    };
复制代码
```

### 5. 注册Clang插件

在Clang插件源码中编写注册代码。编译器会在编译过程中从动态库加载Clang插件。使用FrontendPluginRegistry::Add<>在库中注册插件。注册Clang插件的代码如下：

```
static FrontendPluginRegistry::Add<MSKJPlugin::MSKJASTAction> X("MSKJPlugin", "The MSKJPlugin is my first clang-plugin.");
复制代码
```

在Clang插件代码的最下面，定义的MSKJPlugin字符串是命令行字符串，供以后调用时使用，The MSKJPlugin is my first clang-plugin是对Clang插件的描述。

### 6. 使用clang 插件

利用CMake命令重新生成Xcode工程，可在Loadable modules下看到MSKJPlugin：



![img](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2019/7/24/16c2321405ed2a03~tplv-t2oaga2asx-zoom-in-crop-mark:1304:0:0:0.awebp)



选择MSKJPlugin这个target进行编译，编译完会生成一个动态库文件。

LLVM官方有一个完整可用的Clang插件示例，可以帮我们打印出最上层函数的名字。

通过学习这个插件示例，看看如何使用Clang插件。

使用Clang插件可以通过-load命令行选项加载包含插件注册表的动态库，-load命令行会加载已经注册了的所有Clang插件。使用-plugin选项选择要运行的Clang插件。Clang插件的其他参数通过-plugin-arg-来传递。

cc1进程类似一种预处理，这种预处理会发生在编译之前。cc1和Clang driver是两个单独的实体，cc1负责前端预处理，Clang driver则主要负责管理编译任务调度，每个编译任务都会接受cc1前端预处理的参数，然后进行调整。

有两个方法可以让-load 和-plugin等选项到Clang的cc1进程中：

- 直接使用-cc1选项，缺点是要在命令行上指定完整的系统路径配置；
- 使用-Xclang来为cc1进程添加这些选项。-Xclang参数只运行预处理器，直接将后面参数传递给cc1进程，而不影响clang driver的工作。

下面是一个编译Clang插件，然后使用-Xclang加载使用Clang插件的例子：

```
$ export BD=/path/to/build/directory 
$ (cd $BD && make PrintFunctionNames ) 
$ clang++ -D_GNU_SOURCE -D_DEBUG -D__STDC_CONSTANT_MACROS \
          -D__STDC_FORMAT_MACROS -D__STDC_LIMIT_MACROS -D_GNU_SOURCE \ 
          -I$BD/tools/clang/include -Itools/clang/include -I$BD/include -Iinclude \                                        tools/clang/tools/clang-check/ClangCheck.cpp -fsyntax-only \
          -Xclang -load -Xclang $BD/lib/PrintFunctionNames.so -Xclang \
          -plugin -Xclang print-fns
复制代码
```

上面命令中，先设置构建的路径，再通过make命令进行编译生成PrintFunctionNames.so，最后使用clang命令配合-Xclang参数加载使用Clang插件。

你也可以直接使用-cc1参数，但是就需要按照下面的方式来指定完整的文件路径：

```
$ clang -cc1 -load ../../Debug+Asserts/lib/libPrintFunctionNames.dylib -plugin print-fns some-input-file.c
复制代码
```

### 7. 更多

实现更复杂的插件功能，可以利用clang的API对语法树进行相应的分析与处理。

关于AST的资料：

[clang.llvm.org/doxygen/nam…](https://link.juejin.cn?target=https%3A%2F%2Fclang.llvm.org%2Fdoxygen%2Fnamespaceclang.html)

[clang.llvm.org/doxygen/cla…](https://link.juejin.cn?target=https%3A%2F%2Fclang.llvm.org%2Fdoxygen%2Fclassclang11Decl.html)

[clang.llvm.org/doxygen/cla…](https://link.juejin.cn?target=https%3A%2F%2Fclang.llvm.org%2Fdoxygen%2Fclassclang11Stmt.html)

Clang插件本身的编写和使用并不复杂，关键是如何更好地应用到工作中，通过Clang插件不光能够检查代 码规范，还能够进行无用代码分析、自动埋点打桩、线下测试分析、方法名混淆等。

