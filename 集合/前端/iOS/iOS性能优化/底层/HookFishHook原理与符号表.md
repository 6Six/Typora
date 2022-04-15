# [Hook/FishHook原理与符号表](Hook/FishHook原理与符号表)



## 前言 :

本篇文章较与依赖前一篇 [Mach-O文件](https://juejin.cn/post/6844903983841214472) 的先导知识 , 建议先阅读后再探究 .

> - 由于逆向过程中代码注入往往会使用 `hook` 这种方式 , 而且在安全防护与监测方面经常使用 .
> - 另外只知道 `runtime` 交换 `imp` 的方式对于中高级开发人员  ( 想偷懒又想装* ) 显然是不太够的 .
>
> 那么我们今天就来好好探讨一下 `Hook` 与 `fishHook` 原理 .

## Hook 概述

**HOOK**，中文译为 `“挂钩“` 或 `“钩子”` 。在 **iOS** 逆向中是指改变程序运行流程的一种技术。通过 `hook` 可以让别人的程序执行自己所写的代码。在逆向中经常使用这种技术。所以在学习过程中，我们重点要了解其原理，这样能够对恶意代码进行有效的防护。

大名鼎鼎的 **Hook** 已经不知道被多少人玩出了花 ❀ , 其用途之多我们就不说了 . 

### iOS 中几种常见的 Hook

#### 1 . Method Swizzle

利用 **OC** 的 `Runtime` 特性，动态改变 `SEL`（方法编号）和 `IMP`（方法实现）的对应关系，达到 **OC** 方法调用流程改变的目的。主要用于 **OC** 方法。

常用的有

- `method_exchangeImplementations` 交换函数 **imp**
- `class_replaceMethod` 替换方法
- `method_getImplementation` 与 `method_setImplementation` 直接 `get` / `set` **imp**

> 关于这些 `Runtime` 方法的基本使用以及原理 这一点在 [重签应用调试与代码修改](https://juejin.cn/post/6844903978229039117#heading-14) 这篇文章最后有详细的解释和 `demo` , 感兴趣的可以去阅读一下 .

#### 2 . fishhook

它是 `Facebook` 提供的一个动态修改链接 `Mach-O` 文件的工具。利用 `MachO` 文件加载原理，通过修改懒加载和非懒加载两个表的指针达到 **C** 函数 `Hook` 的目的。

#### 3. Cydia Substrate

`Cydia Substrate` 原名为 `Mobile Substrate` ，它的主要作用是针对 **OC** 方法、**C** 函数以及函数地址进行 `Hook` 操作。当然它并不是仅仅针对 **iOS** 而设计的，安卓一样可以用。官方地址：[www.cydiasubstrate.com/](https://link.juejin.cn?target=http%3A%2F%2Fwww.cydiasubstrate.com%2F)

> 它使用的是 `logos` 语法 , 关于这个工具的使用 , 后续文章我会详细讲述 .

## fishhook 基本使用

### 下载

git - 地址 : [fishhook - git](https://link.juejin.cn?target=https%3A%2F%2Fgithub.com%2Ffacebook%2Ffishhook)

有需要的可以下载这个有中文注释的版本 [link](https://link.juejin.cn?target=https%3A%2F%2Fpan.baidu.com%2Fs%2F1Ma6ufNumi9ZQtWDNFv-lgQ) 提取码:f4f8  .

### demo

```
#import "ViewController.h"
#import "fishhook.h"

@interface ViewController ()
@end

@implementation ViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    
    //rebinding结构体
    struct rebinding nslog;
    //需要HOOK的函数名称，C字符串
    nslog.name = "NSLog";
    //新函数的地址
    nslog.replacement = myNslog;
    //原始函数地址的指针！
    nslog.replaced = (void *)&sys_nslog;
    //rebinding结构体数组
    struct rebinding rebs[1] = {nslog};
    /**
     * 参数1 : 存放rebinding结构体的数组
     * 参数2 : 数组的长度
     */
    rebind_symbols(rebs, 1);
}
//---------------------------------更改NSLog-----------
//函数指针
static void(*sys_nslog)(NSString * format,...);
//定义一个新的函数
void myNslog(NSString * format,...){
    format = [format stringByAppendingString:@"勾上了！\n"];
    //调用原始的
    sys_nslog(format);
}

-(void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event
{
    NSLog(@"点击了屏幕！！");
}
@end
复制代码
```

点击屏幕 , 打印结果 :

```
001--fishHookDemo[15776:645816] 点击了屏幕！！勾上了！
复制代码
```

### 关键函数

`rebind_symbols` ,  源码如下 :

```
int rebind_symbols(struct rebinding rebindings[], size_t rebindings_nel) {
    //prepend_rebindings的函数会将整个 rebindings 数组添加到 _rebindings_head 这个链表的头部
    //Fishhook采用链表的方式来存储每一次调用rebind_symbols传入的参数，每次调用，就会在链表的头部插入一个节点，链表的头部是：_rebindings_head
    int retval = prepend_rebindings(&_rebindings_head, rebindings, rebindings_nel);
    //根据上面的prepend_rebinding来做判断，如果小于0的话，直接返回一个错误码回去
    if (retval < 0) {
    return retval;
  }
    //根据_rebindings_head->next是否为空判断是不是第一次调用。
  if (!_rebindings_head->next) {
      //第一次调用的话，调用_dyld_register_func_for_add_image注册监听方法.
      //已经被dyld加载的image会立刻进入回调。
      //之后的image会在dyld装载的时候触发回调。
    _dyld_register_func_for_add_image(_rebind_symbols_for_image);
  } else {
      //遍历已经加载的image，进行hook
    uint32_t c = _dyld_image_count();
    for (uint32_t i = 0; i < c; i++) {
      _rebind_symbols_for_image(_dyld_get_image_header(i), _dyld_get_image_vmaddr_slide(i));
    }
  }
  return retval;
}
复制代码
```

`fishhook` 的基础使用非常简单.

- 就是需要我们定义一个指针从而 `fishhook` 可以帮我们保存原系统函数的实现地址 ,  另外将需要替换的`函数名称`和`自定义函数地址`写成结构体调用 `rebind_symbols` 就可以了 ,
- 另外可以一次往数组中写入多个结构体进行多个函数的 `hook`.

## fishhook 分析

基础 `OC` 函数的 `hook` 原理我们不在多赘述了 , 其实简单来说就是替换掉方法实现的 `imp` , 这个基于 `OC` 语言的动态运行时机制是很好理解的 .

但是 `C` 呢 ?

> 我们知道 `C` 函数是静态的，也就是说在编译的时候，编译器就知道了它的实现地址，这也是为什么 `C` 函数只写函数声明调用时会报错。那么为什么 `fishhook` 还能够改变 `C` 函数的调用呢？难道函数也有动态的特性存在？我们一起来探究它的原理

### 注意 :

`fishhook` 是可以 `hook` 系统的函数 , 并非所有的 `C` 函数 , 也就是说 **`fishhook` 也只能对带有符号表的系统函数进行重绑定 , 而对自己实现的 C 函数同样是没有办法的.**

我们大可以自己写一个 `C` 函数实验一下 .

```
#import "ViewController.h"
#import "fishhook.h"

@interface ViewController ()
@end

@implementation ViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    
    // hook 自定义C函数
    struct rebinding Cfunction;
    Cfunction.name = "func";
    Cfunction.replacement = newfunc;
    Cfunction.replaced = (void *)&funcOri;
    struct rebinding resbs[1] = {Cfunction};
    rebind_symbols(resbs, 1);
}

// 要hook 的c函数
void func(const char * str){
    NSLog(@"%s",str);
}

//原始的函数指针记录
static void(*funcOri)(const char *);

void newfunc(const char * str){
    NSLog(@"勾住了!");
    funcOri(str);
}
@end
复制代码
```

运行 , 打印结果

```
2019-11-12 14:54:19.001680+0800 fishhookDemo[35238:1563336] 点击了屏幕
2019-11-12 14:54:19.706819+0800 fishhookDemo[35238:1563336] 点击了屏幕
2019-11-12 14:54:19.861428+0800 fishhookDemo[35238:1563336] 点击了屏幕
复制代码
```

无法 `hook` 自定义 `C` 函数 , 接下来我们通过 `fishhook` 原理来解析一下原因 .

### fishhook 原理

首先 :

| C                  | OC                 |
| ------------------ | ------------------ |
| 静态               | 动态               |
| 编译时确定函数地址 | 运行时确定函数地址 |

而系统的 `C` 函数有动态的部分 , 就是我们经常提到的符号表 , 用到的技术叫做 **Position Independent Code** ( 位置代码独立 )  , `fishhook` 也就是在此处做了文章 .

#### `fishhook` 原文讲述 :



![img](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2019/11/12/16e5e71ec6808500~tplv-t2oaga2asx-watermark.awebp)





![img](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2019/11/12/16e5e726720abc2c~tplv-t2oaga2asx-watermark.awebp)



#### 原理概述

由于 `iOS` 系统中 `UIKit` / `Foundation` 库每个应用都会通过 `dyld` 加载到内存中 , 因此 , 为了节约空间 , 苹果将这些系统库放在了一个地方 : **动态库共享缓存区 (dyld shared cache)** .   ( Mac OS 一样有 )  .

因此 , 类似 `NSLog` 的函数实现地址 , 并不会也不可能会在我们自己的工程的 `Mach-O` 中 , 那么我们的工程想要调用 `NSLog` 方法 , 如何能找到其真实的实现地址呢 ?

其流程如下 :

- **在工程编译时 , 所产生的 `Mach-O` 可执行文件中会预留出一段空间 , 这个空间其实就是符号表 , 存放在 `_DATA` 数据段中  ( 因为 `_DATA` 段在运行时是可读可写的 )** 
- 编译时 : **工程中所有引用了共享缓存区中的系统库方法 , 其指向的地址设置成符号地址 ,  ( 例如工程中有一个 `NSLog` , 那么编译时就会在 `Mach-O` 中创建一个 `NSLog` 的符号 , 工程中的 `NSLog` 就指向这个符号 )** 
- 运行时 : **当 `dyld`将应用加载到内存中时 , 根据 `load commands` 中列出的需要加载哪些库文件 , 去做绑定的操作  ( 以 `NSLog` 为例 , `dyld` 就会去找到 `Foundation` 中 `NSLog` 的真实地址写到 `_DATA` 段的符号表中 `NSLog` 的符号上面 )** 

这个过程被称为 `PIC` 技术 . ( Position Independent Code : 位置代码独立 ) 

那么了解了系统函数的整个加载过程 , 我们再来看 `fishhook` 的函数名称 :

`rebind_symbols :: 重绑定符号` 也就简单明了了.

其原理就是 :

> **将编译后系统库函数所指向的符号 , 在运行时重绑定到用户指定的函数地址 , 然后将原系统函数的真实地址赋值到用户指定的指针上.**

那么再回头看自定义的C函数为什么 `hook` 不了 ?

那答案就很简单了 :

- 自定义 `C` 函数实际地址就在自己的 `Mach-O` 内 , 也并没有符号和绑定的过程 .
- 编译时就已经确定了 , 并没有办法操作 .

#### Mach-O 中查看符号表

利用 `MachOView` 直接查看.

![img](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2019/11/12/16e5ea5f7ff86369~tplv-t2oaga2asx-watermark.awebp)



有同学说了 , 说是这么说 , 怎么验证呢 ?

既然讲到了这儿 , 那我们就顺道提一点符号表的知识 , 毕竟一些 bug 收集工具也经常用到符号表还原 , 顺便我们来实际操作一下 , 一边验证理论 , 一边加深记忆 .

## 符号表与实际操作验证理论

从 `MachOView` 中我们看到 , 符号表分为两种

- `Lazy Symbol Pointers`
- `Non-Lazy Symbol Pointers`

就是字面意思 , 懒加载和非懒加载 .

因此我们在使用 `fishhook` 的时候 , 最好是调用一下原函数 , 以防止可能会出现没使用并没有绑定的问题 . 

那么接下来 , 我们来玩一下 ?

废话不多说 , 来到我们刚刚写的 `hook` 了 `NSLog` 的 `demo` , 在 `viewdidload` 中先加一句 `NSLog(@"123")`;

开始玩

### 1 准备好代码和断点



![img](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2019/11/12/16e5ebad1195ad79~tplv-t2oaga2asx-watermark.awebp)



### 2 MachOView 查看

`cmd + r` 运行运行工程来到断点 , 找到 `Mach-O` , 使用 `MachOView` 查看.

![img](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2019/11/12/16e5ebf8c4933f48~tplv-t2oaga2asx-watermark.awebp)



### 3 计算符号地址



![img](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2019/11/12/16e5ec1606321050~tplv-t2oaga2asx-watermark.awebp)



- 首先我们看到 这个符号基于 `Mach-O` 文件的首地址偏移量是 `3028` ,  ( 每个人的都不一样 , 你用你自己的 )  .

  那么 `Mach-O` 的地址在哪呢 ?

  来到工程 `LLDB` 输入指令 : `image list`

  第一个就是我们工程的 `Mach-O` 实际内存地址

  ![img](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2019/11/12/16e5ec6849b5afa1~tplv-t2oaga2asx-watermark.awebp)

  

- 打开计算器 `cmd + 3` , 选择十六进制 , `cmd + v` 把 `Mach-O` 实际内存地址粘贴进去 , 加上 `MachOView` 的符号偏移地址 `3028`.

  ![img](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2019/11/12/16e5ecb0835c0d4a~tplv-t2oaga2asx-watermark.awebp)

  

  `cmd` + `c` 拷贝计算结果.

### 4 lldb 查看内存与汇编代码

`x` + `0x1042C8028` ( 你的计算结果 )

( `memory read` 也可以 , `x` 就是 `memory read` 的简写 )



![img](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2019/11/12/16e5ece27e075bb4~tplv-t2oaga2asx-watermark.awebp)



### 5 查看前八个字节的内容

注意 : `iOS` 小端模式 , 从右往左读 .

那么我上图中对应的实际地址就是 `0x01042c69c0` .

查看汇编 : dis -s 0x01042c69c0 ( 你自己的地址 )

![img](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2019/11/12/16e5ed36f6818831~tplv-t2oaga2asx-watermark.awebp)



那么我们看到里面并没有什么内容 , 也就是说在此时这个断点这里 , 符号并没有被绑定内容 .

过掉断点 , 来到第二处断点 .

( 有对于汇编不熟悉的同学不用着急 , 对比一下第二个断点的结果来看 . 另外后续笔者会考虑继续更汇编部分内容 )

### 重新查看符号

`x` + `0x1042C8028` ( 你的计算结果 )



![img](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2019/11/12/16e5ed767e2af3dc~tplv-t2oaga2asx-watermark.awebp)



可以看到明显内容已经改变了 .

再次查看汇编 : dis -s 0x01042c69c0 ( 你自己的地址 )



![img](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2019/11/12/16e5ed885f582447~tplv-t2oaga2asx-watermark.awebp)



大功告成 ,  再回想一下我们之前的原理探究部分. 完全验证 !

别着急 , 这只是验证了 `iOS` 的 `PIC` 部分 , 那么我们 `fishhook` 呢 ?

- 在 `touchesBegan` 加个断点 ( 不一定非要在 `touchesBegan` 加 , 我这里只是 `fishhook` 的 `rebind_symbols` 后面就没有代码可以过断点了. )
- 过掉当前断点 ( `rebind_symbols` ) , 点击屏幕 来到下一个断点.
- 再次查看内存和汇编

结果如下 :



![img](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2019/11/12/16e5eddffaaf5aa5~tplv-t2oaga2asx-watermark.awebp)



`fishhook` 重绑定后符号指向了我们自定义的函数地址 . 完全验证之前假设 .


作者：李斌同学
链接：https://juejin.cn/post/6844903992904908814
来源：稀土掘金
著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。