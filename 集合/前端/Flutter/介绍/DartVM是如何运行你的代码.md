# [DartVM是如何运行你的代码的](https://cloud.tencent.com/developer/article/1614400)



`Dart VM`有多种方式去运行`Dart`代码，比如：

- `JIT`模式运行源码或者`Kernal binary`
- 通过`snapshot`方式：`AOT snapshot` 和 `AppJIT shanpshot`

两者的主要区别在于`VM`将`Dart`源码转换成可执行代码的时机和方式。

![img](https://ask.qcloudimg.com/http-save/7151929/60yta6ict7.png?imageView2/2/w/1620)isolates

`VM`中的任何`Dart`代码都是运行在隔离的`isolate`当中，`isolate`具有自己的内存(堆)和线程控制的隔离运行环境。`VM`可以同时具有多个`isolate`执行`Dart`的代码，但不同的`isolate`之间不能直接共享任何的状态，只能通过消息端口来进行通信。

我们所说的线程和`isolate`之间的关系其实有点模糊，而且`isolate`也比较依赖`VM`是怎样嵌入到应用程序当中的。线程和`isolate`有下面这样的一些限制：

- 一个线程一次只能进入一个`isolate`，如果线程要进入另一个`isolate`就要先退出当前的
- 一次只能有一个`mutator`线程和`isolate`相关联，`mutator`是用来执行`Dart`代码和调用`VM API`的线程

所以一个线程只能进入一个`isolate`执行`Dart`代码，退出之后才能进入另一个`isolate`。

不同的线程也能进入同一个`isolate`，但不能同时。

当然除了拥有一个`mutator`线程之外，`isolate`还可以有多个`helper`线程，比如：

- 后台`JIT`编译线程
- `GC`线程
- 并发的`GC`标记线程

`VM`内部使用了线程池来管理系统的线程，而且内部是基于`ThreadPool::Task`的概念去构建的，而不是直接使用的系统线程。例如，`GC`的过程就是生成一个`SweeperTask`丢给`VM`的线程池去处理，而不是使用一个专门的线程来做垃圾回收，线程池可以选择一个空闲的线程或者在没有空闲线程的时候新建一个线程来处理这个任务。类似的，消息循环的处理也并没有使用一个专门的`event loop`线程，而是在有新消息的时候产生一个`MessageHandlerTask`给线程池。

## 执行源码

你可以在命令行下直接给`Dart`的源码去执行，例如：

```js
// hello.dart
main() => print('Hello, World!');

$ dart hello.dart
Hello, World!
```

事实上`Dart 2 VM`之后就不再支持直接运行`Dart`源码了，`VM`使用了一种`Kernel binaries(也就是 dill 文件)`包含了序列化的`Kernel ASTs`。所以源代码要先经过通用前端`CFE`处理成`Kernel AST`，而`CFE`是用`Dart`写的，可以给`VM/dart2js/Dart Dev Compiler`这些不同的`Dart`工具使用。

### 前端编译

![img](https://ask.qcloudimg.com/http-save/7151929/kebpauzxsa.png?imageView2/2/w/1620)dart-to-kernel

那么为了保持直接执行`Dart`源码的便捷性，所以有一个叫做`kernel service`的`isolate`，负责将`Dart`源码处理成`Kernel`，`VM`再将`Kernel binary`拿去运行。

![img](https://ask.qcloudimg.com/http-save/7151929/yudrykcsn8.png?imageView2/2/w/1620)kernel-service

但是`CFE`和用户的`Dart`代码是可以在不同的设备上执行，例如在`Flutter`当中，就是将`Dart`代码编译成`Kernel`，和执行`Kernel`的过程个隔离开来，编译`Dart`源码的步骤放在了用户的开发机上，执行`Kernel`放在了移动设备上，`Flutter tools`负责从开发机上将`Kernel binary`发送到移动设备上。

![img](https://ask.qcloudimg.com/http-save/7151929/ixpot58flo.png?imageView2/2/w/1620)flutter-cfe

`flutter tool`并不能自己解析`Dart`源码，它使用了一个叫`frontend_server`的处理，`frontend_server`实际上就是`CFE`的封装和`Flutter`上特定的`Kernel-to-Kernel`的转换。`frontend_server`编译`Dart`源码到`Kernel`文件，`flutter tools`将它同步到执行设备上。`Flutter`的`hot reload`也正是依赖`frontend_server`的，`frontend_server`在`hot reload`的过程中能够重用之前编译中的`CFE`状态，只重编已经更改了的部分。

### Kernel binary 装载

只有`Kernel binary`能够被`VM`加载，并解析创建各种对象。不过这个过程是懒加载的，只有被使用到的库和类的信息才会被装载。每一个程序的实体都会保留指向对应`Kernel binary`的指针，在需要的时候可以去加载更多的信息。

![img](https://ask.qcloudimg.com/http-save/7151929/3qhgtj8v68.png?imageView2/2/w/1620)kernel-loaded-1

类的信息只有在被使用的过程中(例如：查找类的成员，或新建对象)才会被完全反序列化出来，从`Kernel binary`读取类的成员信息，但是函数只会反序列化出函数签名信息，函数体只有在被调用运行的时候才会进一步反序列化出来。

![img](https://ask.qcloudimg.com/http-save/7151929/od7f8fmgym.png?imageView2/2/w/1620)kernel-loaded-2

从`Kernel binary`加载了足够多的信息供运行时成功解析和调用方法之后，就会去解析和调用到`main`函数了。

### 函数编译

程序运行的最初所有的函数主体都不是实际可执行的代码，而是一个占位符，指向`LazyCompileStub`，它只是简单的要求运行时系统为当前的函数生成可执行的代码，然后尾部调用新生成的代码。

![img](https://ask.qcloudimg.com/http-save/7151929/2n50bq4iou.png?imageView2/2/w/1620)raw-function-lazy-compile

首次编译函数时，是通过未优化编译器来完成的。

![img](https://ask.qcloudimg.com/http-save/7151929/6i2m6sbnyn.png?imageView2/2/w/1620)unoptimized-compilation

未优化编译器通过两个步骤来生成机器码：

1. 对函数主体的序列化`AST`进行遍历，以生成函数主体的控制流程图`CFG`。`CFG`由填充了中间语言`IL`指令的基本块组成。这里使用的`IL`指令类似于基于堆栈的虚拟机的指令：从堆栈中获取操作数，执行操作，然后将结果压入同一堆栈。
2. `CFG`使用一对多的低级`IL`指令直接生成机器码：每条`IL`指令扩展为多条机器指令

这个过程中还没有执行优化，未优化编译器的目标是快速的生成可执行指令。这也意味着不会尝试静态解析任何未从`Kernel binary`文件中加载的调用，所以调用的编译是动态完成的。`VM`在这个时候还不能使用任何基于`vitual table`或者`interface table`的调度，而是使用`inline caching`实现动态调用的。

`inline caching`的核心是在调用的时候缓存对应方法解析的结果，`VM`使用的`inline caching`机制包括：

- 一个调用的特殊缓存，将接收的类映射到方法，如果接收者具有匹配的类型则调用方法，缓存还会有一些辅助信息，比如：调用频次计数器，跟踪特定类型出现的频次。
- 一个共享的`stub`，实现方法调用的快速路径，`stub`在给定的缓存中查找是否有和接收者匹配的类型，如果找到了增加相应的频次计数器，并且尾部调用缓存的方法；否则，`stub`调用系统的查找解析逻辑，如果解析成功就更新缓存，并且后续的调用使用对应缓存的方法。

下图说明了`inline cache`在`animal.toFace()`调用时的关系和状态，使用`Dog`实例调用两次，`Cat`实例调用一次：

![img](https://ask.qcloudimg.com/http-save/7151929/i8u4rlptjl.png?imageView2/2/w/1620)inline-cache-1

未优化的编译器足以执行所有的`Dart`代码，只是它的执行速度会慢一些，所以呢`VM`还需要实现自适应的优化编译路径，自适应的优化是采用程序运行时的信息去驱动优化策略。未优化的代码在运行时会收集以下信息：

- `Inline caches`过程中每一个方法调用接受的类型信息
- 执行计数器收集的热点代码区

当某个函数的执行计数器达到某个阈值，这个函数就会提交给后台优化编译器进行优化。

### 优化编译

优化编译的方式和未优化编译有点类似，通过遍历序列化的`Kernel AST`为正在优化的函数构建未优化的`IL`，不同的是与其直接将`IL`转换为机器码，优化编译器会将未优化的`IL`转换成基于`static single assignment (SSA)`的优化`IL`。基于`SSA`的`IL`根据收集到的类型信息，经典的优化手段和`Dart`的特殊优化：比如，`inlining, range analysis, type propagation, representation selection, store-to-load and load-to-load forwarding, global value numbering, allocation sinking, etc`。最后，使用线性扫描寄存器分配器和简单的一对多的`IL`指令，将优化的`IL`降低为机器码。

编译完成之后后端编译器请求`mutator`线程进入一个安全点(`safepoint`)并且将优化的代码`attaches`到对应的调用函数上，下次调用该函数的时候就能直接使用优化的代码。

![img](https://ask.qcloudimg.com/http-save/7151929/hwklb0vz2b.png?imageView2/2/w/1620)optimizing-compilation

需要注意的是，由优化编译器生成的代码是基于运行时收集到的特定信息完成的，例如一个接受动态类型的函数调用，只接收到某个特定的类型，就会被转换成直接的调用，然后检查接收到的类型是否一致。但是在程序的执行过程中，有可能接收到的类型是其他的。

```js
void printAnimal(obj) {
  print('Animal {');
  print('  ${obj.toString()}');
  print('}');
}

// Call printAnimal(...) a lot of times with an intance of Cat.
// As a result printAnimal(...) will be optimized under the
// assumption that obj is always a Cat.
for (var i = 0; i < 50000; i++)
  printAnimal(Cat());

// Now call printAnimal(...) with a Dog - optimized version
// can not handle such an object, because it was
// compiled under assumption that obj is always a Cat.
// This leads to deoptimization.
printAnimal(Dog());
```

### 反优化

优化代码是基于运行时信息对输入做了一些假设而产生的，如果在后续的运行过程中输入和假设不匹配，它就要防止违反这些假设，并且能够在违反的情况能够恢复正常运行。这个过程就叫着反优化：只要优化版本遇到无法处理的情况，它就会将执行转移到未优化函数的匹配点并继续运行。未优化的版本不做任何假设，可以处理所有可能的输入。

`VM`通常会在反优化后放弃优化的版本，然后在以后使用更新的类型反馈再次对其进行优化。`VM`防止违反优化假设一般有两种方式：

- `Inline checks (e.g. CheckSmi, CheckClass IL instructions)`验证输入是否符合优化。例如，将动态调用转换为直接调用时，编译器会在直接调用之前添加这些检查。在此类检查中发生的反优化称为`eager deoptimization`，因为它很容易在 check 的时候被检测出来。
- 全局保护程序，指令运行时在更改优化代码所依赖的内容时丢弃优化代码。例如，优化编译器可能发现某些类C从未扩展过，并在类型传播过程中使用了此信息。但是，随后的动态代码加载或类最终确定可能会引入C的子类-使得假设无效。这个时候，运行时需要查找并丢弃所有在C没有子类的假设下编译的优化代码。运行时可能会在执行堆栈上找到一些现在无效的优化代码，在这种情况下，受影响的`frames`将被标记，并且在执行返回时将对其进行反优化。这种反优化也称为延迟反优化：因为它会延迟到控制权返回到优化代码为止。

## 运行 Snapshots

`VM`有能力序列化`isolate`堆上的对象为二进制的`snapshot`文件，并且可以使用`snapshot`重新创建相同状态的`isolate`.

![img](https://ask.qcloudimg.com/http-save/7151929/kk32g1y51.png?imageView2/2/w/1620)snapshot

`snapshot`针对启动速度做了相应的优化，本质上是要创建的对象的列表和他们之间关系。相对于解析`Dart`源码并逐步创建`VM`内部的数据结构，`VM`可以将`isolate`所必须的数据结构全部打包在`snapshot`中。

但最初`snapshot`是不包括机器码的，在后来开发`AOT`编译的时候就加上去了，开发`AOT`编译和带机器码的`snapshot`是为了允许`VM`在一些无法`JIT`的平台上运行。带代码的`snapshot`几乎和普通的`snapshot`的工作方式是一样的，只是它带有一个代码块，这部分是不需要反序列化的，代码块可以直接`map`进堆内存。

![img](https://ask.qcloudimg.com/http-save/7151929/xdq7o9punt.png?imageView2/2/w/1620)snapshot-with-code

## 运行 AppJIT snapshots

`AppJIT snapshot`可以减少大型`Dart`应用(比如：dartanalyzer 或者 dart2js)的`JIT`预热时间，在小型应用和`VM`使用`JIT`编译的时间差不多。

`AppJIT snapshots`其实是`VM`使用一些模拟的数据来训练程序，然后将生成的代码和`VM`内部的数据结构序列化而生成的，然后分发这个`snapshot`而不是源码或者`Kernel binary`。`VM`使用这个`snapshot`仍然可以在实际运行的过程中发现数据不匹配训练时而启用`JIT`。

![img](https://ask.qcloudimg.com/http-save/7151929/xdq7o9punt.png?imageView2/2/w/1620)

## 运行 AppAOT snapshots

`AOT snapshot`最初是为了无法进行`JIT`编译的平台而引入的，但也可以用来优化启动速度。无法进行`JIT`就意味着：

1. `AOT snapshot`必须包含在应用程序执行期间可以调用的每个功能的可执行代码
2. 可执行代码不能基于运行时的数据进行任何的假设

为了满足这些要求，`AOT`编译过程中会进行全局静态分析(type flow analysis or TFA)，以从已知的入口点确定应用程序的哪些部分是被使用的，分配了哪些类以及类型是如何在程序中传递的。所有这些分析都是保守的，因为必须要保证正确性，有可能会牺牲一点性能，这跟`JIT`不太一样，`JIT`生成的代码还可以通过反优化来回到未优化的代码上运行。然后所有可达的代码块都将被编译成机器码，不会再进行任何的类型推测的优化。编译完所有的代码块之后，就可以获得堆的快照了。

然后，可以使用预编译的运行时来运行生成的`snapshot`，该运行时是`Dart VM`的特殊变体，其中不包括诸如`JIT`和动态代码加载工具之类的组件。

![img](https://ask.qcloudimg.com/http-save/7151929/rww97qincd.png?imageView2/2/w/1620)aot

### Switchable Calls

即使进行了全局和局部分析，`AOT`编译的代码仍可能包含无法静态虚拟化的调用操作。为了弥补这种情况，运行时使用了类似`JIT`过程中的`inline cache`，在这里叫着`switchable calls`.

`JIT`部分上面讲过了，`inline cache`主要包括两部分，一个缓存对象(通常是RawICData)和一个`VM`的调用(例如：InlineCacheStub)，在`JIT`模式下运行时只会更新 cache 的缓存，但是在`AOT`中，运行时可以根据`inline cache`的状态选择替换缓存和要调用的`VM`函数路径。

![img](https://ask.qcloudimg.com/http-save/7151929/epng2hjsoi.png?imageView2/2/w/1620)aot-ic-unlinked

所有的动态调用最初都是`unlinked`状态，首次调用时会触发`UnlinkedCallStub`的调用，它又会调用`DRT_UnlinkedCall`去 link 当前的调用点。

如果`DRT_UnlinkedCall`尝试将调用点的状态切换为`monomorphic`，在这个状态下调用就会被替换成直接调用，它通过一个特殊的入口进入方法，并且在入口处验证类型。

![img](https://ask.qcloudimg.com/http-save/7151929/huhevkx13f.png?imageView2/2/w/1620)aot-ic-monomorphic

在上图的例子中，当 obj.method() 首次执行时，obj 是 C 的实例，那么 obj.method 就会被解析成 C.method，下一次出现同样的调用就会直接调用到 C.method，跳过方法查找的过程。但是进入 C.method 仍然是通过一个特殊的入口进入的，验证 obj 是 C 的实例；如果不是的话，`DRT_MonomorphicMiss`就会被调用尝试去进入下一个状态。C.method 有可能仍然是调用的目标函数，例如，obj 是类D的实例，D继承C并且没有`override`C.method。在这种情况下，我们检查是否可以进入`single target`状态，由`SingleTargetCallStub`实现。

![img](https://ask.qcloudimg.com/http-save/7151929/qu7rferriq.png?imageView2/2/w/1620)aot-ic-singletarget

在`AOT`编译过程中，大部分类会在继承结构的深度优先遍历过程分配一个 ID，如果类C具有D0..Dn这些子类，而且都没有`override C.method`，那么`C.:cid <= classId(obj) <= max(D0.:cid, ..., Dn.:cid)`表示 obj.method 会被解析成 C.method。在这种情况下，与其进行单态类(`monomorphic`状态)的比较，我们可以使用类的 ID 范围去检查C的所有子类。

换言之，调用的时候会使用线行扫描`inline cache`, 类似`JIT`模式(可以查看`ICCallThroughCodeStub, RawICData and DRT_MegamorphicCacheMissHandler`)

![img](https://ask.qcloudimg.com/http-save/7151929/2er5149okp.png?imageView2/2/w/1620)aot-ic-linear

当然，如果线性数组中的检查数量超过阈值，将切换为使用类似字典的数据结构。

![img](https://ask.qcloudimg.com/http-save/7151929/lr1xswdll3.png?imageView2/2/w/1620)aot-ic-dictionary