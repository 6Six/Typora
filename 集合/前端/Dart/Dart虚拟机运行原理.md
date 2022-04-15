# [Dart虚拟机运行原理](http://gityuan.com/2019/10/05/dart_vm/)

## 一、Dart虚拟机

#### 1.1 引言

Dart VM是一种虚拟机，为高级编程语言Dart提供执行环境，但这并意味着Dart在D虚拟机上执行时，总是采用解释执行或者JIT编译。 例如还可以使用Dart虚拟机的AOT管道将Dart代码编译为机器代码，然后运行在Dart虚拟机的精简版环境，称之为预编译运行时(precompiled runtime)环境，该环境不包含任何编译器组件，且无法动态加载Dart源代码。

#### 1.2 虚拟机如何运行Dart代码

Dart VM有多钟方式来执行代码：

- 源码或者Kernel二进制(JIT)
- snapshot
  - AOT snapshot
  - AppJIT snapshot

区别主要在于什么时机以及如何将Dart代码转换为可执行的代码。

#### 1.3 Isolate组成

先来看看dart虚拟机中isolate的组成：

![img](http://gityuan.com/img/dart_vm/vm_run/isolates.png)

- isolate堆是运该isolate中代码分配的所有对象的GC管理的内存存储；
- vm isolate是一个伪isolate，里面包含不可变对象，比如null，true，false；
- isolate堆能引用vm isolate堆中的对象，但vm isolate不能引用isolate堆；
- isolate彼此之间不能相互引用
- 每个isolate都有一个执行dart代码的Mutator thread，一个处理虚拟机内部任务(比如GC, JIT等)的helper thread；

isolate拥有内存堆和控制线程，虚拟机中可以有很多isolate，但彼此之间不能直接状态，只能通过dart特有的端口；isolate除了拥有一个mutator控制线程，还有一些其他辅助线程：

- 后台JIT编译线程；
- GC清理线程；
- GC并发标记线程；

线程和isolate的关系是什么呢？

- 同一个线程在同一时间只能进入一个isolate，当需要进入另一个isolate则必须先退出当前的isolate；
- 一次只能有一个Mutator线程关联对应的isolate，Mutator线程是执行Dart代码并使用虚拟机的公共的C语言API的线程

#### 1.4 ThreadPool组成

虚拟机采用线程池的方式来管理线程，定义在runtime/vm/thread_pool.h

![img](http://gityuan.com/img/dart_vm/vm_run/ThreadPool.png)

ThreadPool的核心成员变量：

- all_workers_：记录所有的workers；
- idle_workers：_记录所有空闲的workers;
- count_started_：记录该线程池的历史累计启动workers个数;
- count_stopped_：记录该线程池的历史累计关闭workers个数；
- count_running_：记录该线程池当前正在运行的worker个数；
- count_idle_：记录该线程池当前处于空闲的worker个数，也就是idle_workers的长度；

ThreadPool核心方法：

- Run(Task*): 执行count_running_加1，并将Task设置到该Worker，
  - 当idle_workers_为空，则创建新的Worker并添加到all_workers_队列头部，count_started_加1；
  - 当idle_workers_不为空，则取走idle_workers_队列头部的Worker，count_idle_减1；
- Shutdown(): 将all_workers_和idle_workers_队列置为NULL，并将count_running_和count_idle_清零，将关闭的all_workers_个数累加到count_stopped_；
- SetIdleLocked(Worker*)：将该Worker添加到idle_workers_队列头部，count_idle_加1， count_running_减1;
- ReleaseIdleWorker(Worker*)：从all_workers_和idle_workers_队列中移除该Worker，count_idle_减1，count_stopped_加1；

对应关系图：

|                     | count_started_   | count_stopped_    | count_running_ | count_idle_      |
| :------------------ | :--------------- | :---------------- | :------------- | :--------------- |
| Run()               | +1(无空闲worker) |                   | +1             | -1(有空闲worker) |
| Shutdown()          |                  | +all_workers_个数 | 清零           | 清零             |
| SetIdleLocked()     |                  |                   | -1             | +1               |
| ReleaseIdleWorker() |                  | +1                |                | -1               |

可见，count_started_ - count_stopped_ = count_running_ + count_idle_；

## 二、JIT运行模式

#### 2.1 CFE前端编译器

看看dart是如何直接理解并执行dart源码

```
// gityuan.dart
main() => print('Hello Gityuan!');

//dart位于flutter/bin/cache/dart-sdk/bin/dart
$ dart gityuan.dart
Hello, World!
```

说明：

- Dart虚拟机并不能直接从Dart源码执行，而是执行dill二进制文件，该二进制文件包括序列化的Kernel AST(抽象语法树)。
- Dart Kernel是一种从Dart中衍生而来的高级语言，设计之初用于程序分析与转换(transformations)的中间产物，可用于代码生成与后端编译器，该kernel语言有一个内存表示，可以序列化为二进制或文本。
- 将Dart转换为Kernel AST的是CFE(common front-end）通用前端编译器。
- 生成的Kernel AST可交由Dart VM、dev_compiler以及dart2js等各种Dart工具直接使用。

![img](http://gityuan.com/img/dart_vm/vm_run/dart-to-kernel.png)

#### 2.2 kernel service

有一个辅助类isolate叫作kernel service，其核心工作就是CFE，将dart转为Kernel二进制，然后VM可直接使用Kernel二进制运行在主isolate里面运行。

![img](http://gityuan.com/img/dart_vm/vm_run/kernel-service.png)

#### 2.3 debug运行

将dart代码转换为kernel二进制和执行kernel二进制，这两个过程也可以分离开来，在两个不同的机器执行，比如host机器执行编译，移动设备执行kernel文件。

![img](http://gityuan.com/img/dart_vm/vm_run/flutter-cfe.png)

图解：

- 这个编译过程并不是flutter tools自身完成，而是交给另一个进程frontend_server来执行，它包括CFE和一些flutter专有的kernel转换器。
- hot reload：热重载机制正是依赖这一点，frontend_server重用上一次编译中的CFE状态，只重新编译实际更改的部分。

#### 2.4 RawClass内部结构

虚拟机内部对象的命名约定：使用C++定义的，其名称在头文件raw_object.h中以Raw开头，比如RawClass是描述Dart类的VM对象，RawField是描述Dart类中的Dart字段的VM对象。

1）将内核二进制文件加载到VM后，将对其进行解析以创建表示各种程序实体的对象。这里采用了懒加载模式，一开始只有库和类的基本信息被加载，内核二进制文件中的每一个实体都会保留指向该二进制文件的指针，以便后续可根据需要加载更多信息。

![img](http://gityuan.com/img/dart_vm/vm_run/kernel-loaded-1.png)

2）仅在以后需要运行时，才完全反序列化有关类的信息。（例如查找类的成员变量，创建类的实例对象等），便会从内核二进制文件中读取类的成员信息。 但功能完整的主体(FunctionNode)在此阶段并不会反序列化，而只是获取其签名。

![img](http://gityuan.com/img/dart_vm/vm_run/kernel-loaded-2.png)

到此，已从内核二进制文件加载了足够的信息以供运行时成功解析和调用的方法。

所有函数的主体都具有占位符code_，而不是实际的可执行代码：它们指向LazyCompileStub，该Stub只是简单地要求系统Runtime为当前函数生成可执行代码，然后对这些新生成的代码进行尾部调用。

![img](http://gityuan.com/img/dart_vm/vm_run/raw-function-lazy-compile.png)

#### 2.5 查看Kernel文件格式

gen_kernel.dart利用CFE将Dart源码编译为kernel binary文件(也就是dill)，可利用dump_kernel.dart能反解kernel binary文件，命令如下所示：

```Java
//将hello.dart编译成hello.dill
$ cd <FLUTTER_ENGINE_ROOT>
$ dart third_party/dart/pkg/vm/bin/gen_kernel.dart          \
       --platform out/android_debug/vm_platform_strong.dill \
       -o hello.dill                                        \
       hello.dart

//转储AST的文本表示形式
$ dart third_party/dart/pkg/vm/bin/dump_kernel.dart hello.dill hello.kernel.txt
```

gen_kernel.dart文件，需要平台dill文件，这是一个包括所有核心库(dart:core, dart:async等)的AST的kernel binary文件。如果Dart SDK已经编译过，可直接使用out/ReleaseX64/vm_platform_strong.dill，否则需要使用compile_platform.dart来生成平台dill文件，如下命令:

```Java
//根据给定的库列表，来生成platform和outline文件
$ cd <FLUTTER_ENGINE_ROOT>
$ dart third_party/dart/pkg/front_end/tool/_fasta/compile_platform.dart \
       dart:core                                                        \        
       third_party/dart/sdk/lib/libraries.json                          \
       vm_outline.dill vm_platform.dill vm_outline.dill                 
```

#### 2.6 未优化编译器

首次编译函数时，这是通过未优化编译器来完成的。

![img](http://gityuan.com/img/dart_vm/vm_run/unoptimized-compilation.png)

未优化的编译器分两步生成机器代码：

- AST -> CFG: 对函数主体的序列化AST进行遍历，以生成函数主体的控制流程图(CFG)，CFG是由填充中间语言（IL）指令的基本块组成。此阶段使用的IL指令类似于基于堆栈的虚拟机的指令：它们从堆栈中获取操作数，执行操作，然后将结果压入同一堆栈
- IL -> 机器指令：使用一对多的IL指令，将生成的CFG直接编译为机器代码：每个IL指令扩展为多条机器指令。

在此阶段没有执行优化，未优化编译器的主要目标是快速生成可执行代码。

#### 2.7 内联缓存

未优化编译过程，编译器不会尝试静态解析任何未在Kernel二进制文件中解析的调用，因此（MethodInvocation或PropertyGet AST节点）的调用被编译为完全动态的。虚拟机当前不使用任何形式的基于虚拟表(virtual table)或接口表(interface table)的调度，而是使用内联缓存实现动态调用。

虚拟机的内联缓存的核心思想是缓存方法解析后的站点结果信息，对于内联缓存最初是为了解决函数的本地代码：

- 站点调用的特定缓存(RawICData对象)将接受者的类映射到方法，缓存中记录着一些辅助信息，比如方法和基本块的调用频次计数器，该计数器记录着被跟踪类的调用频次；
- 共享的查找存根，用于实现方法调用的快速路径。该存根在给定的高速缓存中进行搜索，以查看其是否包含与接收者的类别匹配的条目。 如果找到该条目，则存根将增加频率计数器和尾部调用缓存的方法。否则，存根将调用系统Runtime来解析方法实现的逻辑，如果方法解析成功，则将更新缓存，并且随后的调用无需进入系统Runtime。

![img](http://gityuan.com/img/dart_vm/vm_run/inline-cache-1.png)

#### 2.8 编译优化

未优化编译器产生的代码执行比较慢，需要自适应优化，通过profile配置文件来驱动优化策略。内联优化，当与某个功能关联的执行计数器达到某个阈值时，该功能将提交给后台优化编译器进行优化。

优化编译的方式与未优化编译的方式相同：通过序列化内核AST来构建未优化的IL。但是，优化编译器不是直接将IL编译为机器码，而是将未优化的IL转换为基于静态单分配（SSA）形式的优化的IL。

对基于SSA的IL通过基于收集到的类型反馈，内联，范围分析，类型传播，表示选择，存储到加载，加载到加载转发，全局值编号，分配接收等一系列经典和Dart特定的优化来进行专业化推测。最后，使用线性扫描寄存器分配器和一个简单的一对多的IL指令。优化编译完成后，后台编译器会请求mutator线程输入安全点，并将优化的代码附加到该函数。下次调用该函数时，它将使用优化的代码。

![img](http://gityuan.com/img/dart_vm/vm_run/optimizing-compilation.png)

另外，有些函数包含很长的运行循环，因此在函数仍在运行时将执行从未优化的代码切换到优化的代码是有意义的，此过程之所以称为“堆栈替换”（OSR）。

VM还具有可用于控制JIT并使其转储IL以及用于JIT正在编译的功能的机器代码的标志

```Java
$ dart --print-flow-graph-optimized         \
       --disassemble-optimized              \
       --print-flow-graph-filter=myFunc     \
       --no-background-compilation          \
       hel.dart
```

#### 2.9 反优化

优化是基于统计的，可能出现违反优化的情况

```Java
void printAnimal(obj) {
  print('Animal {');
  print('  ${obj.toString()}');
  print('}');
}

// 大量调用的情况下，会推测printAnimal假设总是Cat的情况下来优化代码
for (var i = 0; i < 50000; i++)
  printAnimal(Cat());

// 此处出现的是Dog，优化版本失效，则触发反优化
printAnimal(Dog());
```

每当只要优化版本遇到无法解决的情况，它就会将执行转移到未优化功能的匹配点，然后继续执行，这个恢复过程称为去优化：未优化的功能版本不做任何假设，可以处理所有可能的输入。

虚拟机通常会在执行一次反优化后，放弃该功能的优化版本，然后在以后使用优化的类型反馈再次对其进行重新优化。虚拟机保护编译器进行推测性假设的方式有两种：

- 内联检查（例如CheckSmi，CheckClass IL指令），以验证假设是否在编译器做出此假设的使用场所成立。例如，将动态调用转换为直接调用时，编译器会在直接调用之前添加这些检查。 在此类检查中发生的取消优化称为“急切优化”，因为它在达到检查时就急于发生。
- 运行时在更改优化代码所依赖的内容时，将会丢弃优化代码。例如，优化编译器可能会发现某些类从未扩展过，并且在类型传播过程中使用了此信息。 但是，随后的动态代码加载或类最终确定可能会引入C的子类，导致假设无效。此时，运行时需要查找并丢弃所有在C没有子类的假设下编译的优化代码。 运行时可能会在执行堆栈上找到一些现在无效的优化代码，在这种情况下，受影响的帧将被标记为不优化，并且当执行返回时将进行不优化。 这种取消优化称为延迟取消优化，因为它会延迟到控制权返回到优化代码为止。

## 三、Snapshots运行模式

### 3.1 通过Snapshots运行

1）虚拟机有能力将isolate的堆（驻留在堆上的对象图）序列化成二进制的快照，启动虚拟机isolate的时候可以从快照中重新创建相同的状态。

![img](http://gityuan.com/img/dart_vm/vm_run/snapshot.png)

Snapshot的格式是低级的，并且针对快速启动进行了优化，本质上是要创建的对象列表以及如何将它们连接在一起的说明。那是快照背后的原始思想：代替解析Dart源码并逐步创建虚拟机内部的数据结构，这样虚拟机通过快照中的所有必要数据结构来快速启动isolate。

2）最初，快照不包括机器代码，但是后来在开发AOT编译器时添加了此功能。开发AOT编译器和带代码快照的动机是为了允许虚拟机在由于平台级别限制而无法进行JIT的平台上使用。

带代码的快照的工作方式几乎与普通快照相同，只是有一点点不同：它们包括一个代码部分，该部分与快照的其余部分不同，不需要反序列化。该代码节的放置方式使其可以在映射到内存后直接成为堆的一部分

![img](http://gityuan.com/img/dart_vm/vm_run/snapshot-with-code.png)

### 3.2 通过AppJIT Snapshots运行

引入AppJIT快照可减少大型Dart应用程序（如dartanalyzer或dart2js）的JIT预热时间。当这些工具用于小型项目时，它们花费的实际时间与VM花费的JIT编译这些应用程序的时间一样多。

AppJIT快照可以解决此问题：可以使用一些模拟训练数据在VM上运行应用程序，然后将所有生成的代码和VM内部数据结构序列化为AppJIT快照。然后可以分发此快照，而不是以源（或内核二进制）形式分发应用程序。如果出现实际数据上的执行配置文件与培训期间观察到的执行配置文件不匹配，快照开始的VM仍可以采用JIT模式执行。

### 3.3 通过AppAOT Snapshots运行

AOT快照最初是为无法进行JIT编译的平台引入的，对于无法进行JIT意味着：

- AOT快照必须包含应用程序执行期间可能调用的每个功能的可执行代码;
- 可执行代码不得依赖于执行期间可能违反的任何推测性假设

为了满足这些要求，AOT编译过程会进行全局静态分析（类型流分析, TFA），以确定从已知入口点集中可访问应用程序的哪些部分，分配了哪些类的实例以及类型如何在程序中流动。 所有这些分析都是保守的：这意味着它们会在正确性方面出错，与可以在性能方面出错的JIT形成鲜明对比，因为它始终可以取消优化为未优化的代码以实现正确的行为。

然后，所有可能达到的功能都将编译为本地代码，而无需进行任何推测性优化。但是，类型流信息仍用于专门化代码（例如，取消虚拟化调用），编译完所有函数后，即可获取堆的快照。

最终的快照snapshot可以运行在预编译Runtime，该Runtime是Dart VM的特殊变体，其中不包括诸如JIT和动态代码加载工具之类的组件。

![img](http://gityuan.com/img/dart_vm/vm_run/aot.png)

AOT编译工具没有包含进Dart SDK。

```Java
//需要构建正常的dart可执行文件和运行AOT代码的runtime
$ tool/build.py -m release -a x64 runtime dart_precompiled_runtime

// 使用AOT编译器来编译APP
$ pkg/vm/tool/precompiler2 hello.dart hello.aot

//执行AOT快照
$ out/ReleaseX64/dart_precompiled_runtime hello.aot
Hello, World!
```

#### 3.3.1 Switchable Calls

1）即使进行了全局和局部分析，AOT编译的代码仍可能包含无法静态的去虚拟化的调用站点。为了补偿此AOT编译代码和运行时，采用JIT中使用的内联缓存技术的扩展。此扩展版本称为可切换呼叫 （Switchable Calls）。

JIT部分已经描述过，与调用站点关联的每个内联缓存均由两部分组成：一个缓存对象（由RawICData实例表示）和一个要调用的本机代码块（例如InlineCacheStub）。在JIT模式下，运行时只会更新缓存本身。但在AOT运行时中，可以根据内联缓存的状态选择同时替换缓存和要调用的本机代码。

![img](http://gityuan.com/img/dart_vm/vm_run/aot-ic-unlinked.png)

最初，所有动态呼叫均以未链接状态开始。首次调用此类呼叫站点时，将调用UnlinkedCallStub，它只是调用运行时帮助程序DRT_UnlinkedCall来链接此呼叫站点。

2）如果可能，DRT_UnlinkedCall尝试将呼叫站点转换为单态状态。在这种状态下，呼叫站点变成直接呼叫，该呼叫通过特殊的单态入口点进入方法，该入口点验证接收方是否具有预期的类。

![img](http://gityuan.com/img/dart_vm/vm_run/aot-ic-monomorphic.png)

在上面的示例中，假设第一次执行obj.method（）时，obj是C的实例，而obj.method则解析为C.method。

下次执行相同的调用站点时，它将直接调用C.method，从而绕过任何类型的方法查找过程。但是，它将通过特殊的入口点(已验证obj仍然是C的实例)进入C.method。如果不是这种情况，将调用DRT_MonomorphicMiss并将尝试选择下一个调用站点状态。

3）C.method可能仍然是调用的有效目标，例如obj是C的扩展类但不覆盖C.method的D类的实例。在这种情况下，检查呼叫站点是否可以转换为由SingleTargetCallStub实现的单个目标状态（见RawSingleTargetCache）。

![img](http://gityuan.com/img/dart_vm/vm_run/aot-ic-singletarget.png)

此存根基于以下事实：对于AOT编译，大多数类都使用继承层次结构的深度优先遍历来分配整数ID。如果C是具有D0，…，Dn子类的基类，并且没有一个覆盖C.method，则C.:cid <= classId（obj）<= max（D0.:cid，…，Dn .:cid）表示obj.method解析为C.method。在这种情况下，我们可以将类ID范围检查（单个目标状态）用于C的所有子类，而不是与单个类（单态）进行比较

否则，呼叫站点将切换为使用线性搜索内联缓存，类似于在JIT模式下使用的缓存。

![img](http://gityuan.com/img/dart_vm/vm_run/aot-ic-linear.png)

最后，如果线性数组中的检查数量超过阈值，则呼叫站点将切换为使用类似字典的结构

![img](http://gityuan.com/img/dart_vm/vm_run/aot-ic-dictionary.png)

## 四、附录

#### 4.1 源码说明

整个过程相关的核心源码，简要说明：

- runtime/vm/isolate.h： isolate对象
- runtime/vm/thread.h：与连接到isolate对象的线程关联的状态
- runtime/vm/heap/heap.h： isolate的堆
- raw_object.h： 虚拟机内部对象
- ast.dart：定义描述内核AST的类
- kernel_loader.cc：其中的LoadEntireProgram()方法用于将内核AST反序列化为相应虚拟机对象的入口点
- kernel_service.dart：实现了Kernel Service isolate
- kernel_isolate.cc：将Dart实现粘合到VM的其余部分
- pkg/front_end：用于解析Dart源码和构建内核AST
- pkg/vm: 托管了大多数基于内核的VM特定功能，例如各种内核到内核的转换；由于历史原因，某些转换还位于pkg/kernel;
- runtime/vm/compiler: 编译器源码
- runtime/vm/compiler/jit/compiler.cc：编译管道入口点
- runtime/vm/compiler/backend/il.h： IL的定义
- runtime/vm/compiler/frontend/kernel_binary_flowgraph.cc： 其中的BuildGraph(), 内核到IL的翻译开始，处理各种人工功能的IL的构建
- runtime/vm/stub_code_x64.cc：其中StubCode::GenerateNArgsCheckInlineCacheStub()，为内联缓存存根生成机器代码
- runtime/vm/runtime_entry.cc：其中InlineCacheMissHandler()处理IC没有命中的情况
- runtime/vm/compiler/compiler_pass.cc: 定义优化编译器的遍历及其顺序
- runtime/vm/compiler/jit/jit_call_specializer.h：进行大多数基于类型反馈的专业化
- runtime/vm/deopt_instructions.cc： 反优化过程
- runtime/vm/clustered_snapshot.cc：处理快照的序列化和反序列化。API函数家族Dart_CreateXyzSnapshot[AsAssembly]负责写出堆的快照，比如 Dart_CreateAppJITSnapshotAsBlobs 和Dart_CreateAppAOTSnapshotAsAssembly。
- runtime/vm/dart_api_impl.cc： 其中Dart_CreateIsolate可以选择获取快照数据以开始isolate。
- pkg/vm/lib/transformations/type_flow/transformer.dart: TFA（类型流分析）以及基于TFA结果转换的切入点
- runtime/vm/compiler/aot/precompiler.cc：其中Precompiler::DoCompileAll()是整个AOT编译的切入点

#### 4.2 参考资料

https://mrale.ph/dartvm/