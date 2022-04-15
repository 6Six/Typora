

# [美团外卖Flutter动态化实践](https://tech.meituan.com/2020/06/23/meituan-flutter-flap.html)

## 一、前言

Flutter 跨端技术一经推出便在业内赢得了不错的口碑，它在“多端一致”和“渲染性能”上的优势让其他跨端方案很难比拟。虽然 Flutter 的成长曲线和未来前景看起来都很好，但不可否认的是，目前 Flutter 仍处在发展阶段，很多大型互联网企业都无法毫无顾虑地让全线 App 接入，而其中最主要的顾虑是包大小与动态化。

动态化代表着更短的需求上线路径，代表着大大压缩了原始包的大小，从而获得更高的用户下载意向，也代表着更健全的线上质量维护体系。当明白这些意义后，我们也就不难理解，在 Flutter 的应用与适配趋近完善时，动态化自然就成为了一个无法避开的话题。RN 和 Weex 等成熟技术甚至让大家认为动态化是跨端技术的标配。

美团外卖 MTFlutter 团队从 2019 年 9 月开始对动态化进行研究，目前已在多个业务模块上线，内部项目代号 “Flap” 。。

## 二、Flap 的特点与优势

Flap 研发的初心是为了提供一个完整解决方案，而不是一个过渡方案。项目组思考了当下最痛的点并逐一列出，然后再根据目标来做具体选型。在前期，只有需求考虑得越周全，后续的架构和研发才会越明确。在研发过程中，团队应该坚守底线，坚守初心，不断攻克困难，完成昔日定下的目标。

### 2.1 核心目标

- 通用性，保持 Flutter 多平台支持的能力且方案无平台差异。
- 低成本，动态化对齐 Flutter 生态和常规开发习惯，且可低成本转化现有的 Flutter 页面。
- 适用性，避免包过大、不稳定等不利于应用的缺陷。
- 高性能，保留 Flutter 渲染性能极佳的特点。

### 2.2 动态化选型

**a. 产物替换**

选型中首先考虑到的是下发产物替换，官方在也曾经推出了 Code Push 方案，甚至可以支持 Diff 差量下载，但是在 2019 年 4 月被叫停，这里引用一下官方的发言 [Flutter/issues/14330](https://github.com/flutter/flutter/issues/14330)：

> To comply with our understanding of store policies on Android and iOS, any solution would be limited to JIT code on Android and interpreted code on iOS. We are not confident that the performance characteristics of such a solution on iOS would reach the quality that we demand of our product. (In other words, “it would be too slow”.)
>
> There are some serious security concerns. Since these patches would essentially allow arbitrary code execution, they would be extremely attractive malware vectors. We could mitigate this by requiring that patches be signed using the same key as the original package, but this is error prone and any mistake would have serious consequences. This is, fundamentally, the same problem that has plagued platforms that allow execution of code from third-party sources. This problem could be mitigated by integrating with a platform update mechanism, but this defeats the purpose of an out-of-band patching mechanism.

简而言之，就是官方对动态化后的性能没有自信，并且对安全性有所顾虑。之前，官方提供方案的局限性也十分明显。比如对 Native-Flutter 混合 App 支持不友好，并且无法进行灰度等业务定制操作，所以不能满足通用性和高性能的核心目标。

**b. AOT 搭载 JIT**

Flutter 在 Release 模式下构建的是 AOT 编译产物，iOS 是 AOT Assembly，Android 默认 AOTBlob。 同时 Flutter 也支持 [JIT Release](https://github.com/flutter/flutter/wiki/JIT-Release-Modes) 模式，可以动态加载 Kernel snapshot 或 App-JIT snapshot。如果在 AOT 上支持 JIT，就可以实现动态化能力。但问题在于，AOT 依赖的 Dart VM 和 JIT 并不一样，AOT 需要一个编译后的 “Dart VM”（更准确地说是 Precompiled Runtime），JIT 依赖的是 Dart VM（一个虚拟机，提供语言执行环境）；并且 JIT Release 并不支持 iOS 设备，构建的应用也不能在 AppStore 上发布。

实现此方案需要抽离一份 DartVM 独立编译，再以动态库的形式引入项目。通过初步测试，发现会增大包体积 20MB+，这超过了 MTFlutter 之前做 Flutter 包体积优化的总和。进一步让 Flutter 包体积成为推广与接入业务方的巨大阻碍，不满足我们对适用性的要求。

**c. 动态生产 DSL**

Native 侧本身具备 JS 动态执行环境，利用这个执行环境动态生成包含页面和逻辑事件绑定 DSL，进而解析为 Flutter 页面或组件，也可以实现动态化诉求。技术思路接近 RN，但与其不同的是利用 Flutter 渲染引擎和框架。这种先将代码执行起来再获取 DSL 的手段，我们简称为动态生产 DSL。

此方案可以很好地支持逻辑动态化，但弊端也比较明显。首先要对齐 Flutter 框架，JS 侧的开发量很大且开发体验受损。另外，对 JS 的依赖偏重，构建的 JS 框架本身解释执行有一定开销，对于页面逻辑与事件在运行中需要频繁地进行 Flutter 与 JS 的跨平台通信，同样也会产生一定开销。这不能满足 MTFlutter 团队对高性能的诉求。更严重的是，此方案对开发同学的开发习惯并不友好，将 Dart 改为 JS，现有的 Flutter 开发工具无法直接使用，这与低成本诉求背道而驰。

**d. 静态生产 DSL**

前面说 “将代码执行起来再获取 DSL 的手段，我们简称为动态生产 DSL”，那么代码不执行直接转换 DSL，就称为静态生产 DSL 方案。

静态生产的特点是抹平了平台差异，因为 input 是 Dart source 与平台无关，直接将 Dart source 内的完整信息通过一层转换器转换到 DSL，然后通过 Native 和 Dart 的静态映射和基础的逻辑支持环境，使得其可以在纯 Dart 的环境下渲染与交互。

在具体实现上，可以利用 Dart-lang 官方提供的 Analyzer 分析库（该工具在 Dartfmt、Dart Doc、Dart Analyzer Server 中都有使用）构建 DSL。该库提供了一组 API 能对 Dart source 进行分析，按照文件粒度生成 AST 对象。AST 对象用整齐的数据结构包含了 Dart 文件的所有信息，利用这些信息可以便捷地生成所需的 DSL。**所有的这个分析 + 转换的过程全部在线下进行**。接下来， DSL-JSON 以 Zip 的形式下发，Flutter 的 AOT 侧以此为数据源，完成整个 Flutter 项目的渲染与交互。

这种方案，一来可以保持 Flutter/Dart 的开发体验，也没有平台差异，逻辑动态化依赖静态映射和基础逻辑支持，而非 JScore，有效地避免了性能上的开销。综上考虑，静态生产 DSL 最终成为 MTFlutter 团队选型的方案。

### 2.3 项目架构

![图1 Flap整体架构](https://p0.meituan.net/travelcube/2dae09a9a27dc9198828bc63e88fd65088608.png)

图1 Flap整体架构



如图 1 所示，三处浅绿色部分为一个阶段的阶段产物，起到承上启下的作用。以绿色部分为界，整体架构自然而然的就被划分成了三个区域：

- 下层第一部分是对开发阶段的赋能，产物是正确且规范（也满足 Flap 规范）的 Dart 源码。
- 第二部分是 DSL 的转换器，产物是 JSON 格式的 DSL，用于标准化的描述页面层级与逻辑。
- 上层的第三部分是运行时环境，准备了所有需要的符号构建 Dart 对象与逻辑，产物是动态化 App 或动态化的模块。

## 三、Flap 的原理与挑战

图 1 中的核心模块是转换器部分和运行时部分，接下来会介绍下这两个部分的原理与部分实现。

### 3.1 转换器原理

**AST & DSL**

AST 意为抽象语法树（Abstract Syntax Tree）。Dart 的 AST 和其他语言的 AST 基本概念类似。’package:front_end/src/scanner/token.dart’ 中定义了所有的 Token，AST 也是通过词法分析、语法分析、解层级嵌套得到。ASTNode 对象作为存储编译单元中重要信息的基本数据结构，派生类基本分为 Declaration、Expression、Literal、Statement。

DSL 意为领域特定语言（Domain-specific Language）。表示专门针对特定问题领域的编程语言或者规范语言。相对自然语言，编程语言是不灵活的，它的语法和语义设计常取决于它的执行环境和特定目的。过去人们总是发明新的编程语言，近年来新出现的语言越来越相近，因此 DSL 也变得流行起来。

那 Flap 的 DSL 具体是什么？对于开发者而言，那这个 DSL 就是 Dart Code。而对于机器或 App 而言，那这个 DSL 就是 JSON。

前面的技术选型中提到：

> 利用 Dart-lang 官方提供了 Analyzer 分析库，官方的 Analyzer 的能力可以拿来直接用，该库提供了一组 API 能对 Dart source 进行分析，按照文件粒度生成 AST 对象，该数据结构包含了 input 的 Dart 文件的所有信息。

我们的 DSL 的基本原理就是对 AST 内数据的一个描述， 并附带一些其他操作。

![图2 DSL-JSON 的转换步骤](https://p1.meituan.net/travelcube/95f243a1da7dafacffd72732dd758723267968.png)

图2 DSL-JSON 的转换步骤



因为用 Analyzer 的 API 跑出的 AST 也叫 CompilationUnit，实际上是一个编译单元，里面还存有很多编译相关的属性例如 lineInfo、beginToken 等。但使用 DSL 的方式不依赖编译，所以很多不需要的属性会被裁剪或忽略。

在转换器入口会对大类（identifier、statementImpl、literal、methodInvocation 等等）进行分发，每一个大类的数据结构使用一种中间结构 Dart model 来传输，然后对于大类中细分的类型（IfStatement、AssignmentStatement、DoStatement、SwitchStatement 等等），配有足够细粒度的转换接口，以 AST 结构作为输入，以 Map 节点作为输出。最终定义并提炼了 10 种标准的 Map 结构（class、method、variable、stmt 等等）来承载所有类型。

**举个例子**

一个简单的 Widget 节点经过转换后得到这样的 DSL-JSON，可以看到 DSL 的可读性还是 OK 的（默认下发时产物是一个压缩成单行并加密的二进制文件，这里是解密后 Format 换行后展示的）。我们在转换中会区分普通的字符串、变量名引用、系统枚举等类型，加以不同的符号表示。

![图3 常规 Widget 组件的源码与 DSL 示例](https://p0.meituan.net/travelcube/ea4103609d43bb98cc170bcccdb00ce248195.png)

图3 常规 Widget 组件的源码与 DSL 示例



**关于逻辑**

举一个简单的四则运算的例子，可以看出在对于“乘法应当先计算”这个规则上，我们的 DSL 能够自动遵循， 其中的奥秘是 Analyzer 帮我们做了这种运算优先级的判断，归根结底还是一种描述 AST 的工作，我们自己不会去根据静态代码做分析过程。

![图4 简单逻辑的代码与DSL示例](https://p0.meituan.net/travelcube/073d51ceee2016d7aa02418ce6aed18552075.png)

图4 简单逻辑的代码与DSL示例



**关于语法糖**

语法糖往往画风清奇，结构与众不同，但是在 AST 中还是很诚实的，该什么结构就是什么结构。所以语法糖应该在转换器侧进行展开为常规结构再转 DSL，而不是对特殊格式设置特殊的 DSL 传到运行时再去解析。

![图5 部分语法糖的展开情况](https://p1.meituan.net/travelcube/92d97d0bf6a5def90f412ed46370429150331.png)

图5 部分语法糖的展开情况



这里只举了一些简单的例子，只是 DSL 体系中的一个片段，实际在项目落地时有很多较为复杂的逻辑，类似于循环套循环内进行集合操作或是异步回调内加多重三目逻辑等等。这里因为篇幅原因和涉及到业务代码相关就不展开详细的介绍了，其中的原理是一样的，都是描述 AST 的过程中增加一些特殊处理，最终会将转换产物的 Map 节点根据原有 AST 的层级结构组装起来，再通过 JSONEncode 转为 JSON。

![图6 DSL内部结构层级](https://p0.meituan.net/travelcube/cc1764a8594f04cb6511ffbba46becee95272.png)

图6 DSL内部结构层级



转换器侧能够完整的描述一个 Dart 文件的所有信息，如图 6 所示。值得一提的是，不同的节点还可能出现任意结构，method 里的 Argument 里可能是一个全局变量，条件表达式的右边又可能是一个方法。对于这种相同的结构即使出现在不同的位置也应当使用一套处理逻辑来转换，因此转换器是以迭代为主加小范围递归的设计思路。

将细粒度转换接口按照具体类别分在不同文件中（statement_factory、class_factory、function_factory） 等待解析生产总线的调用。实际操作中各个类之间是近似于网状的调用，因此所有调用应当都是 Static 的，并且内部隔离，不引用不修改外部变量，做到无副作用。

DSL 转换器是一个命令行程序，因此可以无缝的部署到自动化的机器上。新代码合入主干后， 接下来的 Bundle 生成与分发逻辑都可以使用各种图形化界面的发布系统来操作。

### 3.2 运行时原理

**Prepare & Running**

运行时相关的操作是在 App 内发生的，包括初始化，拉取 DSL，解析与使用。 简言之可以分为 Prepare 和 Running 两个阶段。Prepare 是准备各种运行时所需的符号，包括系统类符号与自定义符号，属性符号与方法符号（这里所说的符号实际就是 Dart 内的对象）。Prepare 阶段完成才能进行后续的 Running 相关操作，具体是页面的构建，事件的绑定，交互与逻辑的正常运转。

![图7 运行时原理的两大阶段](https://p0.meituan.net/travelcube/69002c34046ede9555a75eef4ba6cdcc106577.png)

图7 运行时原理的两大阶段



**万能方法 Function.apply()**

Flutter 期望线上产品是编译后的“完全体现”，同时为了避免生成过大的包，并不支持 Dart:Mirror。“Flutter apps are pre-compiled for production, and binary size is always a concern with mobile apps, we disabled dart:mirrors.”那么，在这种前提下，如何将外部符号转内部符号？Function() 对象提供了这样一个万能方法。

```dart
// function.dart
external static apply(Function function, List positionalArguments,
      [Map<Symbol, dynamic> namedArguments]);
```

第一个参数是 Function 类型，后两个参数是该函数所需的参数（位置参数与命名参数，这两者在 DSL 中都可以取到），因此只要能获取到某个 Function，那就能在任何时候调用它。

此 Function 若为 Constructor Function 那返回值则为构造出的对象类型。

**Proxy-Mirror**

DSL 后只能得到字符串的标识，因此需要建立一个 String 与 Function 的映射关系，考虑到类名方法名，数据结构应该是 {String:{String:Function}}，通过 className 和 functionName 两个 String Key 即可取得一一对应的 Function()，下面给出一个系统类的类方法（构造方法）的代码片段：

```dart
{
  'EdgeInsets': 
  {
    'fromLTRB': (left, top, right, bottom) => EdgeInsets.fromLTRB(left, top, right, bottom),
    // ...other function
  },
  // ...other class
};
```

然后对于系统类的实例方法、getter、setter 则需要在外部多传一个 instance 参数，instance 是外部通过该类的构造方法的 func 创建后传入。

```dart
// instance method
"inflateSize": (instance, size) => instance.inflateSize(size),
// getter
"horizontal": (instance) => instance.horizontal,
// setter
"last": (List instance, dynamic value) => instance.last = value,
```

**Custom Class’s meta**

对于自定义类，我们需要构建一个模拟的元类系统，存放所有符号信息，在解析时将所有的 JSON 节点转成可处理的对象。所有的属性声明都会构建成 FlapVariable 类型，所有的方法声明都会构建成 FlapFunction 类型。

如图 8 所示，父类和元类也是有相应的指针，父类的成员变量也会填充到子类，并且通过 mixin 的方式将类相关属性注入到派生类类型，例如 FlapState，FlapState 继承自 state，这样既可以让系统类的生命周期方法留个调用链的开口，也可以使用注入的运行时类属性。

![图8 运行时模拟的元类系统](https://p0.meituan.net/travelcube/206a67936ba065070947259fc6bb3f4059320.png)

图8 运行时模拟的元类系统



**Evaluate**

如下面代码的例子，一个 if 语句的 JSON 节点下发后，经过 parser 之后会得到一个 IfStatement 对象，这类对象都有一个特点就是包含几个属性，和一个运行时入口方法 evaluate（Scope scope）。这个方法在抽象类 Evaluative 类中，所有语句和表达式的类都会继承于此，自动获得 evaluate 方法，其中属性部分是在解析过程中解析成 Dart 对象后通过构造方法的参数传入的。

```dart
class IfStatement extends Statement {
  dynamic condition = undefined;
  Body thenBody;
  Body elseBody;
  IfStatement(this.condition, this.thenBody, [this.elseBody]);
  // 简化版代码
  ProcessResult evaluate(Scope scope) {
    bool conditionValue = condition.evaluate(scope)
    if (conditionValue){
      return thenBody(Scope);
    }else{
      return elseBody(Scope);
    }
  }
}
```

属性中的条件对象与语句对象在解析的过程中并不会被触发， 真正的触发是方法被调用时从运行时的入口方法 evaluate 进入，此时才会通过作用域 Scope 判定条件是 true or false，然后调用到其他需要 evaluate 的 Dart 对象，如下图 9 所示：

![图9 运行时 evaluate 触发链路](https://p1.meituan.net/travelcube/49d722548eee1d435332bdb5e819012357180.png)

图9 运行时 evaluate 触发链路



经过表达式的堆叠，实现了语句，经过语句的堆叠实现了 body，再补充上形参和返回值，则就构成了我们运行时中的自定义方法 FlapFunction。这里要用到一下仿真函数的概念，FlapFunction 要实现 call 方法，这样在外部调用时就真的和 Function 画风一致了。

动态化页面运行时，Flap 会维持一套作用域体系。Scope 的结构相当于双向链表，每一个 Scope 有 outer 和 inner 两个指针。全局作用域的 outer 为 null，inner 为类作用域；类作用域的 inner 为局部作用域；局部作用域的 inner 可能为 null 也可能又是一个局部作用域；随便哪一个作用域顺着 outer 一直往上找，肯定能找到全局作用域。

**Scope**

Scope 在逻辑的执行中实际就是充当了 Context 上下文的作用，因为每个方法或表达式被 evalute 时需要一个 Scope 入参，这个 Scope 是从外部传入的，并且这一行语句对象执行后 Scope 还会作为入参传给下一行语句。比如第一行语句声明了一个 “code” 的变量，第二行语句对这个 “code” 进行修改，则需要先通过引用从 Scope 中取出这个 “code” 的值，不但可以从 Scope 中取出声明的属性，也可以取出声明过的方法，方法内也是可以调用方法的。这也就解释了为什么我们可以处理自定义方法中的逻辑。

![图10 Scope的寻找与构建](https://p0.meituan.net/travelcube/4d324a877d9d382da88116ee386dbab6143686.png)

图10 Scope的寻找与构建



图 10 描述了 Scope 在实际运用中的两种场景。左半部分是点击按钮触发 onTap 回调，需要找到 confirm 方法，此时会先从局部作用域的方法列表里找，没找到，则会 outer 一层去类作用域里寻找，此时找到了该方法的实现。

右半部分展示了执行该方法的 body 时是需要传入的 Scope 是如何构建的。先从符号大本营中获取全局变量、全局属性构成全局作用域，再从此类的元类中取出属性和方法构成类作用域，再构建局部作用域，当然参数也是会放到局部作用域里的，以此构建了完整的 Scope 传入 body 的 evaluate 方法支撑后面的逻辑执行。

### 3.3 遇到的挑战

**工作量大，需要长期有耐心**

首先解释下，这里的工作量大并不是指系统方法映射等这种体力活的工作量大，这些我们都是有自动生成且按需生成的（生态部分会提到）。我们所说的工作量大，主要是指涵盖转换器、运行时的研发以及生态相关建设等，我们要尽可能的满足所有的 Dart 语法才能让业务代码能够低成本的转换，并且有众多的脚本与工具支撑。

**项目复杂，需要设计合理的架构以支撑扩展**

在项目的分模块开发中，各个模块（parser、intermediate、runtime 等等）严格遵守单一职责原则与最小知道原则，最大化的杜绝了模块间耦合，模块与模块的通信由一些标准的数据结构进行（map 或继承自 ASTNode 的结构）。 这就使得任何一个模块出现重大重构时不会影响到其他模块，其中底层核心的几个类的单侧覆盖率接近100%，有专人负责优化。并且在项目中随处可以抽象类、接口类、mixin 类等，这也就使得随着支持的能力越来越复杂时，项目的可读性不会成反比，代码不会变“恶心”，而是以整齐的方式扩张，文件多而不乱。

**疑难杂症较多，对问题保持足够的信心**

有时候会遇到一些诸如静态方法调用构造方法时作用域被覆盖、循环语句嵌套时内侧 continue 之后外侧语句也会跟着停、某方法参数的 Function 取完引用之后 Function 也跟着执行了等等的 Bug，解 Bug 是开发中必不可少的一部分，有时候加个 if else 用 easy way 可以很快解决，但我们不会那么做，探索优雅 Right Way 的乐趣是研发过程中的一个重要组成部分。

相比于草草了事之后，每晚睡前都会面临这段代码“灵魂”拷问，我们更愿意多花时间思考把代码写的像 Mac pro 主机的包装那样“丝滑”。这样的工作氛围培养了每位同学的信心，只要是必现问题，基本都能优雅地解决。

## 四、生态支撑

虽然 Flap 的设计理念使得其在开发效率与执行效率上有一定的亮点，但这还不足以让其在业务中快速推广。因此我们建设了一套完整的 Flap 生态体系，涵盖了开发、发布、测试、运维各阶段。

![图11 Flap在美团内网生态](https://p0.meituan.net/travelcube/7810c0364cd2fe099b53e7004587a1a0123200.png)

图11 Flap在美团内网生态



如图 11 所示，Flap 生态的特点可以用 稳、快、准、狠 四个字来表达。

### 4.1 稳

稳，意为可靠的质量管理体系。在①IDE 开发中②提测阶段③线上监控④降级容灾，我们都有对应的策略。其中②和③的基本是和 Native 类似的 PR 检查、QA、日志、上报之类的这里就不做赘述了，下面主要提一下①和④。

**IDE 语法检测插件**

这个功能的意义是尽早地将不支持的语法以编译错误的方式暴露出来，以便同学在开发期就能发现及时修改。 设想一下当你代码写完了，Code Review 也逃过了同学的眼睛，PR 的 Dart 检测也过了，开开心心下班了，突然一个电话打来说发 Bundle 的时候错了，有的语法 Flap 不支持，需要返工去改，此时你的内心一定会“万马奔腾”。

所以，我们将这种暂不支持的语法提前暴露，并推荐使用什么方式代替，可以有效的减少返工， 得到一份满足 Dart 规范和 Flap 规范的代码。同样的 Lint 检测规则后续也配置到了 PR 阶段，如果真出现插件规则更新不及时场景，也会被拦在 PR 阶段。

![图12 IDE语法检测插件](https://p0.meituan.net/travelcube/a1b918f2d6ed7c23063cbe8ecc01694432700.png)

图12 IDE语法检测插件



不过，目前 Flap 不支持的语法已经很少了，目前基本就是 await、as 和超过 2 个 with 等场景， 其中 await 和多个 with 的理论上也能支持，但会让项目有较大的重构和多处的分别对待，不利于后期的维护，考虑到 await 完全可以使用 future.then 代替，所以这个语法就禁了。对于 mixin 的特性，在 Dart 侧本身就是排列组合的关系。超过 2 个 with 会产生多个派生类，动态化的实现类似，所以为了不让简单问题复杂化，我们也禁用了 2 个以上 with 的写法，还有一些写法上的限制，例如 import 不使用全路径也会报错。

目前开发中 Flap 动态化已经与 AOT 共用一份业务代码了，为了不让 Flap 的规则影响到项目中还未覆盖到动态化的页面，让其满屏报错，我们使用 @Flap 注解作为是否开启当前页面的 Flap 规范检测的开关。这也很好理解，当这个页面内没有 @Flap 时，肯定是个 AOT 模块则还是默认的 Dart 检测规则， 一旦加上了 @Flap(‘pageID’)，说明此页面会被动态发版，所以会自动开启 Flap 检测规则。

**降级容灾**

Flap 接入了美团内部统一的动态化发布平台 DD，并利用 DD 平台的能力实现了 App 版本、平台类型、UUID、Flutter SDK 版本等细粒度的下发规则管控。业务方可以根据实际情况选择不同的策略灰度发布方案，如果发生了严重异常，Flap 也支持撤包操作。

![图13 Bundle发布系统的各项边界控制](https://p0.meituan.net/travelcube/ca1f1c611856f03b9d03fa119db93c9c81632.png)

图13 Bundle发布系统的各项边界控制



某一个页面加了标记支持了动态化之后，也会继续进行 AOT 编译过渡2个版本， 前置页面点击跳转是跳 AOT 页还是跳 Flap 页完全由 URL 里的参数控制，这个 URL 不是完全由云端下发的，是代码中先写上默认的 URL，若需要在配置平台修改后，下发的配置信息会让这个 URL 在路由侧完成替换。即使配置平台挂了，顶多丧失 URL 的替换能力而不是无法前往落地页。

![图14 URL 动态替换与条件配置](https://p0.meituan.net/travelcube/2aa168ceda07e5545fce72eb5f5113b070712.png)

图14 URL 动态替换与条件配置



对于 Flap 还有个更犀利的功能，在过渡期间（Flap 已经上线且 AOT 代码还没删时），一旦 Flap 出现 Dart 异常， 当用户退出页面再进入时会自行进入该 pageID 下的 Flutter AOT 页面，最大化降低对用户的干扰。

### 4.2 快

快，意为快速发版，快速更新。Flap 动态化改造使应用具备了分钟级动态发版的能力，为了更全面地释放这个能力，客户端业务迭代的流程也做了相应的调整。

当业务包发版上线，到了应用运行阶段，Flap 主要面对的问题变成敏捷与质量的平衡，即：如何保证动态代码能够尽快生效，同时又要保证加载性能和稳定性。

对于此问题，Flap 的解法是二级缓存与实时更新相结合，线上环境使用内存 + 磁盘二级缓存，进入页面之后再预拉取更新包，平衡加载性能与更新实时性。而线下环境则强制加载远程包，实现测试代码的快速交付。

![图15 Flap二级缓存策略](https://p0.meituan.net/travelcube/a319e781af8ec33d810f14156c23d15b46617.png)

图15 Flap二级缓存策略



得益于这种机制，Flap 在线上可以实现接近 Web 的触达效率：应用会在启动时和具体业务入口处发起更新请求，每当业务有动态发布，新版本页面即可在用户下一次打开时触达至用户。在加载性能方面，二级缓存加持下的页面加载时间仅为数十毫秒，而远程加载的时间也只有 1 秒左右。

### 4.3 准

**细粒度动态化**

准，指哪打哪，可以页面级动态化，也可以局部 Widget 级别的细粒度动态化。事实上在 Flutter 的世界中，“页面”本身也是一个 Widget，业务方在实际开发中，只需要增加一行注解，即可实现对应 Widget 或页面的动态化。

```dart
@Flap('close_protect')
class CloseProtectWidget extends StatelessWidget {
  // ...Widget 的 UI 和逻辑实现
}
```

Flap 打包发版时，解析引擎会从注解标记的 Widget 入手，递归解析所有依赖的文件，转化成对应的 DSL 并打包。App 线上运行时，每个动态化的页面或组件都会按照注解的 FlapId，通过 FlapWidgetContainer 还原成对应的 UI。

![图16 注解的扫描与widget构建](https://p0.meituan.net/travelcube/ee02c0b5f39b48a2552f2936edad7c2c54353.png)

图16 注解的扫描与widget构建



实际调用时，只需传入注解中标记的 FlapId，即可实现动态化区域或页面的加载和渲染。

```dart
// 局部 Widget 级别的动态化，通过 FlapWidgetContainer 加载
Column(
  children: <Widget>[
    MyAOTWidget(),  // 原生 Flutter AOT Widget
    FlapWidgetContainer(widgetId: 'kangaroo_card'), // Flap widget
  ],
);

// 页面级别的动态化，通过 MTFlutterRoute 路由跳转：
RouteUtils.open('scheme://host/mtf?mtf_page=flap&flap_id=close_protect');
```

**精准的 Debug 能力**

在 Debug 阶段加上一个注解 @Flap（‘pageId’），就会自动尝试转 DSL。如果该页面非常独立，且语法没有太花哨，则直接就能看到转换完成的字样。这个就说明该页面用到的语法既支持 Dart 又支持 Flap，不需要做任何修改。如果出现错误，则会在终端下精准打印出错误的位置。在此功能支持之前，基本都是“一崩就崩”到系统类的某某方法，开发同学只能通过自己的经验去堆栈中往上找。目前的精准 Debug 能力实现了转换器、运行时 parser、运行时 evaluate 三个阶段的全面覆盖。

![图17 三个阶段的 Debug 定位](https://p0.meituan.net/travelcube/b51c55ad7b97b8a4d6f989551ccd9a8e269199.png)

图17 三个阶段的 Debug 定位



在转换器阶段的报错位置信息可直接在 Exception 中获得 AST 对象的 lineinfo 进而获取到列号行号信息。在 parser 与 evaluate 阶段的错误定位是根据对核心方法的 trycatch 与设置通用 Exception 类型逐层上抛实现的。因为 DSL-JSON 会被压缩且可以 format，行号列号并无意义，所以在运行时阶段的报错全是精确到某 class 中的某 method。

### 4.4 狠

狠，各种自动生成，实际转换步骤操作方式简单粗暴。Flap 在整个迭代流程环节都提供了便捷的自动化工具支撑。

**imports 自动加载**

基于 Flap 转换一个旧的 Flutter AOT 页面到 Flap 页面的操作是简单粗暴的，加上注解，一行终端指令就可以一把“梭”。但一个业务页面为了设计上的合理往往会分成多个文件，如果有 10 个文件是不是要重复 10 遍这样的工作？答案是否定的。Flap 无论是在 DSL 转换器侧，还是在运行时加载 DSL，都会做到 imports 的递归加载。

IDE 语言检测插件有一条限制是：import 必须使用 package 全路径，不能只 import 一个类名。因为多文件需要导入的位置都是根据全路径截取出的相对路径来计算的。

**Proxy-mirror 按需生成**

前面介绍过 Proxy-Mirror 是外部符号转内部符号的桥梁， 那么具体 Dart 文件中哪些用到的类或方法需要内置 Proxy，而哪些类不需要呢？这个划分的边界就是，在转换的代码内能否看到此类或方法的声明。系统方法的声明肯定不在业务文件里，所以需要 Proxy。业务 Model 的声明在“我的业务”文件中有，所以不需要 Proxy。代码中使用到了官方 Pub 或是其他业务线的 Pub，例如美团金融的 Pub 里的方法，声明不在“我的业务”文件里，所以需要 Proxy。

在 Flutter AOT 迁移动态化初期，经常需要手动干预的问题是：项目中遇到 Proxy-Mirror 缺失会打断转换器， 需要手动补充后继续进行转换。

对于这种问题后期研发 Proxy 自动生成按需生成的工具， 主要原理是在预转换阶段，先扫描代码的 AST Tree，压平层级获取所有的项目结构中 identifer 节点包裹的 Value，进行一系列判定规则，然后基于[reflectable](https://pub.dev/packages/reflectable) 功能实现 Proxy 的自动生成。

**发布链路“一条龙”服务**

经过不断的提炼与简化，目前开发者大可以将注意力集中在开发阶段，一旦代码合入主干，接下来就会有完整的 Flap 工程化发布和托管系统协助开发者完成后续的打包、发布、运维流程。前面介绍过的所有细节工作，都会由这些工具自动化完成，实现便捷发布。Flap 也在路由层面对接了集团内通用的运维工具，开发者无须任何额外操作即可实现加载时间、FPS、异常率等基础指标的监控。对于指标波动、异常升高等情况，也会自动注册报警项并关联至当前的打包人。

## 五、业务实践经验

业务落地只是我们的目标之一，更重要的是在业务实践过程中，发现框架问题，完善各类语法特性支持，提高在复杂的混合场景下的兼容性，反哺促进框架的完善。不断打磨的同时完善工作流，思考与沉淀最佳实践，逐渐总结出合理的调试方案、操作步骤与协作方式，不断提升开发效率与体验。完善动态化基建及工具链建设，完成动态化流程的自动化与工程化，进一步降低转换与开发成本。

### 5.1 应用场景

对于 Flap 在业务中的实践，主要有两种应用场景。

**场景1. 原有 Flutter 页面，需要转换成动态化页面**

设想一下，理想状态下一个好的动态化框架应该是怎样的？动态化框架将原有 Flutter 改写成支持动态化的页面？那加一个 @Flap 注解就好了。然后就可以提交代码，自动走工具链那一套。

目前，虽然没有达到理想状态，但我们也在无限接近中，当然还是要简单地本地调试一下。基本都需要改个 URL 路由和 Mock 环境之类的步骤，我们已经提供了模板的调试工程，支持一键对比 AOT 与动态化运行之后的差异，如图 18 所示。基本就是加上注解，IDE插件会报错哪些语法不支持，需要换一种写法，然后跑一下就可以，然后就提交代码。

![图18 研发过程支持不同的运行模式](https://p1.meituan.net/travelcube/7e27d66f0b6d7818ff73278d70babb29175182.png)

图18 研发过程支持不同的运行模式



**场景2. 直接使用 Flap 技术栈开发新页面**

重新开发场景很明显比第一种要简单，因为没有历史包袱。设想一下，好的动态化框架应该怎么做？就是和 Flutter 的 AOT 开发使用一套相同的 IDE 环境，相同的开发模式，就是 IDE 会多报几项语法错误罢了，开发时就能直接被提示到换一种写法就行。写完后加上注解，然后再提交代码。

### 5.2 实践经验

目前，我们团队已经把 Flutter 动态化能力在一些业务场景落地，当然业界也会有相似的或者不同的动态化方案。无论方案本身怎样， 在落地时的步骤基本都大同小异，我们也总结了一些经验。

**绕过问题并加以记录**

初期任何框架的能力都不是完美的，都会存在问题。业务方同学遇到 Proxy 类缺失之类等比较简单的问题可以直接解决，运行时环境的深层问题、某些语法在复杂叠加场景下出现异常等等，一般会先尝试用其他的语法绕过，记录文档，然后同步到 Flap 团队同学进行解决。

**定时补充 IDE Plugin Rules**

对明确不支持的语法、关键字等添加到 IDE Plugin Rules 中，并提供了相关语法的替代方案，Rules 也会定时补充和删减。

**提前周知各方资源**

包括确认好 Android 的上线节奏，QA 的测试节奏，以及周知PM动态化的覆盖占比。

**关键权限收紧管理**

相比于整理权限、灰度、降级、容灾等线上的 SOP 和 FAQ，让大家都学着操作， 直接指定 2~3 位超级管理员看上去更靠谱，线上环境由“老司机”把控更好。

### 5.3 落地结果

业务应用涵盖 App 一级页在内的多个页面，场景既有页面动态化，也有局部动态化，经受住了一级、二级页面的流量验证。

![图19 部分动态化落地页面](https://p0.meituan.net/travelcube/97b7867c09a82bae1e484f14adb09fc0187471.png)

图19 部分动态化落地页面



![图20 部分动态化页面FPS数据](https://p0.meituan.net/travelcube/a6264f90bd9174e10f64b6204c86abf783162.png)

图20 部分动态化页面FPS数据



![图21 部分动态化页面渲染时长](https://p0.meituan.net/travelcube/0e02ebafc90972f1968506255582f37f127665.png)

图21 部分动态化页面渲染时长



图 21 涉及到 PV 的地方打了马赛克，Flap 团队对包括 FPS、加载时间、Bundle 下载时长、渲染时长等 11 项指标进行了统计，可以看到 FPS 平均是在 58 以上，渲染时长根据页面复杂度的不同在 7~96ms 之间。

总的来说，各项指标表现均接近于 Flutter 原生性能。并且图中的数据都还有可提升的空间，目前的平均值也受到了局部较差数值的影响，后续会根据不同的 TP 分位使用分层的优化方案。

## 六、总结与展望

我们通过静态生产 DSL+Runtime 解释运行的思路，实现了动态下发与解释的逻辑页面一体化的 Flutter 动态化方案，建设了一套 Flap 生态体系，涵盖了开发、发布、测试、运维各阶段。目前 Flap 已在美团多个业务场景落地，大大缩短了需求的发版路径，增强了线上问题修复能力。Flap 的出现让 Flutter 动态化和包大小这两个短板得到了一定程度的弥补，促进了 Flutter 生态的发展。此外，多个技术团队对 Flap 表示出了极大的兴趣，Flap 在更多场景的接入和共建也正在进行中。

未来我们还会进一步完善复杂语法支持能力和生态建设，降低开发和转换 Flap 的成本，提升开发体验，争取覆盖更多业务场景，积极探索与业务方共建。然后基于大前端融合，探索打通其他技术栈，基于Flap DSL 抹平终端差异的可能。

## 参考文献

- [1] Gilad Bracha. The Dart Programming Language [M]. Addison-Wesley Professional，2015
- [2] [Code Push/Hot Update](https://github.com/flutter/flutter/issues/14330)
- [3] [Analyzer 0.39.10](https://pub.dev/packages/analyzer)
- [4] [Extension API](https://code.visualstudio.com/API)
- [5] [Flutter核心技术与实战](https://time.geekbang.org/column/intro/200)
- [6] [App Store Review Guidelines](https://developer.apple.com/app-store/review/guidelines/)