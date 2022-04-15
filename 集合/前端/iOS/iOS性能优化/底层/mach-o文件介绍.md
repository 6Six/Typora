# [Mach-O文件介绍](https://juejin.cn/post/6844903983841214472)



## 前言

前几篇文章中 :

[shell 脚本自动重签名与代码注入](https://juejin.cn/post/6844903976379351053)

[应用签名原理及重签名 (重签微信应用实战)](https://juejin.cn/post/6844903969811070990)

[重签应用调试与代码修改 (Hook)](https://juejin.cn/post/6844903978229039117)

我们对重签名和代码注入有了一定的了解 . 那么这个过程中我们反复提到一个最重要的文件 -- `Mach-O` .

那么说来说去 , 这个`Mach-O` 到底是个什么 . 既然它这么重要 , 那么我们有必要去好好的了解一下它 .

( 对概念不太感兴趣的同学可以直接跳到第二章节 `Mach-O` 的文件结构 ) 

## MachO 文件

`Mach-O` 其实是 `Mach Object` 文件格式的缩写，是 `mac` 以及 `iOS` 上可执行文件的格式， 类似于 `windows` 上的 `PE` 格式 ( Portable Executable ) , `linux` 上的 `elf` 格式 ( Executable and Linking Format )  .

它是一种用于可执行文件、目标代码、动态库的文件格式。作为 `a.out` 格式的替代，`Mach-O` 提供了更强的扩展性。

但是除了可执行文件外 , 其实还有一些文件也是使用的 `Mach-O` 的文件格式 .

属于 `Mach-O` 格式的常见文件

> - 目标文件 .o
> - 库文件
>   - .a
>   - .dylib
>   - Framework
> - 可执行文件
> - dyld ( 动态链接器 )
> - .dsym ( 符号表 )

Tips : 使用 `file` 命令可以查看文件类型

![img](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2019/10/29/16e159769a6a285f~tplv-t2oaga2asx-watermark.awebp)



也就是说 `Mach-O` 并非一定是可执行文件 , 它是一种文件格式 , 分为 `Mach-O Object` 目标文件 、 `Mach-O ececutable` 可执行文件、 `Mach-O dynamically` 动态库文件、 `Mach-O dynamic linker` 动态链接器文件、 `Mach-O dSYM companion` 符号表文件 , 等等 .

大家可以自己通过 `vim` 几个 `.c` , 然后 `clang` 生成 `.o` 目标文件和可执行文件来玩一下 , 以便更好地理解这几种文件以及其编译的模式 .

那么上图中我们还看到一个 `arm64` , 这个是什么意思呢 ?

> - 在 release 模式下
> - 支持 iOS 11.0 系统版本以下

**当满足这两个条件时 , 我们的应用打包出来的 `Mach-O ececutable` 可执行文件是包含 `arm64` 以及 `arm_v7` 的架构的** , `iPhone 5C` 以上机型都是 `64` 位系统了 .

那么包含了支持多架构的 `Mach-O ececutable` 可执行文件被称为 :  **通用二进制文件** , 即多种架构都可读取运行 .

另外 `Xcode` 中通过编译设置 `Architectures` 是可以更改所生成的 `Mach-O ececutable` 可执行文件的支持架构的 .



![img](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2019/10/29/16e166fc843d27d9~tplv-t2oaga2asx-watermark.awebp)



> 编译器在生成 `Mach-O` 文件会选择 `Architectures` 以及 `Valid Architectures` 的交集 , 因此想要支持多架构的话 , 在`Valid Architectures` 中继续添加就可以了 , 编译生成 `Mach-O` 之后 , 使用 `file` 命令可以检查下结果 .

### 通用二进制文件

- 苹果公司提出的一种程序代码。能同时适用多种架构的二进制文件
- 同一个程序包中同时为多种架构提供最理想的性能。
- 因为需要储存多种代码，通用二进制应用程序通常比单一平台二进制的程序要大。
- 但是由于两种架构有共通的非执行资源，所以并不会达到单一版本的两倍之多。
- 而且由于执行中只调用一部分代码，运行起来也不需要额外的内存。

通用二进制文件通常被称为 `Universal binary` , 在 `MachOView` 等 中叫做 `Fat binary` , **这种二进制文件是可以完全拆分开来 , 或者重新组合的** , 那么接下来我们来玩一下 .

### Fat binary 的组合与拆分

#### 1 - 新建工程 , 选择支持系统版本 `10.3` .



![img](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2019/10/29/16e16814b067cdb9~tplv-t2oaga2asx-watermark.awebp)



#### 2 - 编辑运行模式



![img](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2019/10/29/16e1681dde12018a~tplv-t2oaga2asx-watermark.awebp)





![img](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2019/10/29/16e16829d5b0ca34~tplv-t2oaga2asx-watermark.awebp)



选择 `Release`  ( 测试完毕改回来 . 否则 `run` 太慢 )

#### 3 - Build Settings



![img](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2019/10/29/16e168729d111f41~tplv-t2oaga2asx-watermark.awebp)

第一行是自带的环境变量 , 你自己也可以删掉自己写 , `iOS 10.3` 以上 + `release` 环境下会默认包含 `arm64 + armv7` 的架构 , 因此我们自己加上 `armv7s` 和 `arm64e` .



#### 4 - 选择真机 run

`run` 起来后找到 `Mach-O` 文件



![img](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2019/10/29/16e168b404628092~tplv-t2oaga2asx-watermark.awebp)





![img](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2019/10/29/16e168fc69459d5d~tplv-t2oaga2asx-watermark.awebp)



可以看到 , 我们的 `Fat binary` 就已经生成好了 .

使用 `lipo - info` 命令也是可以查看支持架构的

![img](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2019/10/29/16e169855cc80726~tplv-t2oaga2asx-watermark.awebp)



#### 拆分 `Fat binary`

```
lipo macho文件名称 -thin 要拆分哪个架构 -output 拆分出来文件名
复制代码
```

例:

```
lipo 通用二进制MachO_Test -thin armv7s -output macho_armv7s
复制代码
```

然后我们就看到文件夹多了一个 `macho_armv7s` , 查看一下 :



![img](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2019/10/29/16e169e5e17b7ddb~tplv-t2oaga2asx-watermark.awebp)



另外拆分后源文件并不会改变.

#### 合并 `Fat binary`

```
lipo -create macho_arm64 macho_arm64e macho_armv7 macho_armv7s -output newMachO
复制代码
```

合并后我们来看下新生成的 和以前的文件的哈希值 .



![img](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2019/10/29/16e16c0b0c9af80e~tplv-t2oaga2asx-watermark.awebp)



一模一样的 .

Tips:

> **这种方式在我们合并静态库的时候会经常用到** , 因为静态库本身就是 `Mach-O` 文件嘛 , 另外我们在逆向的时候 , 有时也经常会用这种方法拆分二进制文件 , 因为我们只需要分析单一架构即可 , 无须肥大的二进制文件.

#### 补充

另外稍微补充一点 , 多架构二进制文件组合成通用二进制文件时 , 代码部分是不共用的  ( 因为代码的二进制文件不同的组合在不同的 `cpu` 上可能会是不同的意义 )  . 而公共资源文件是会共用的 .

## Mach-O 文件结构



![img](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2019/10/29/16e16fc9c838b80d~tplv-t2oaga2asx-watermark.awebp)



`Mach-O` 的组成结构如图所示包括了

- `Header` 包含该二进制文件的一般信息
  - 字节顺序、架构类型、加载指令的数量等。
  - 使得可以快速确认一些信息，比如当前文件用于 `32` 位还是 `64` 位，对应的处理器是什么、文件类型是什么
- `Load commands` 一张包含很多内容的表
  - 内容包括区域的位置、符号表、动态符号表等。
- `Data` 通常是对象文件中最大的部分
  - 包含 `Segement` 的具体数据

我们来找一个 Mach-O 文件 使用 MachOView 或者 otool 命令去查看一下文件结构 .



![img](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2019/10/30/16e1a7d2e663d73e~tplv-t2oaga2asx-watermark.awebp)





![img](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2019/10/30/16e1a7d9f6191ac2~tplv-t2oaga2asx-watermark.awebp)



那么这个 `Mach-O` 到底这些部分存放的是什么内容 , 加下来我们就来一一探索一下 .

### Mach Header



![img](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2019/10/30/16e1a97cf737816a~tplv-t2oaga2asx-watermark.awebp)



`Header` 中存储的内容大致如上图所示 , 那么每一条到底对应着什么呢 ? , 我们打开源码看一下, `cmd + shift + o` , 搜索 `load.h` , 找 `mach_header_64` 结构体.

```
struct mach_header_64 {
    uint32_t	magic;		/* 魔数,快速定位64位/32位 */
    cpu_type_t	cputype;	/* cpu 类型 比如 ARM */
    cpu_subtype_t	cpusubtype;	/* cpu 具体类型 比如arm64 , armv7 */
    uint32_t	filetype;	/* 文件类型 例如可执行文件 .. */
    uint32_t	ncmds;		/* load commands 加载命令条数 */
    uint32_t	sizeofcmds;	/* load commands 加载命令大小*/
    uint32_t	flags;		/* 标志位标识二进制文件支持的功能 , 主要是和系统加载、链接有关*/
    uint32_t	reserved;	/* reserved , 保留字段 */
};
复制代码
```

`mach_header_64` 相较于 `mach_header` , 也就是 `32` 位头文件 , 只是多了一个保留字段 . `mach_header` 是链接器加载时最先读取的内容 , 它决定了一些基础架构 , 系统类型 , 指令条数等信息.

### Load Commands

`Load Commands` 详细保存着加载指令的内容 , 告诉链接器如何去加载这个 `Mach-O` 文件.

通过查看内存地址我们发现 , 在内存中 , `Load Commands` 是紧跟在 `Mach_header` 之后的 .

那么这些 `Load Commands` 对应了什么呢 ? 我们以 arm64 为例.



![img](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2019/10/30/16e1ac75319c4bd8~tplv-t2oaga2asx-watermark.awebp)



其中 **_TEXT** 段和 **_DATA** 段 , 是我们经常需要研究的 , `MachOView` 下面也有详细列出.



![img](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2019/10/30/16e1ac4c9543cd8a~tplv-t2oaga2asx-watermark.awebp)



### _TEXT 段

我们来看看 `_TEXT` 段里都存放了什么 , 其实真正开始读取就是从 `_TEXT` 段开始读取的 .

| 名称                      | 内容             |
| ------------------------- | ---------------- |
| `_text`                   | 主程序代码       |
| `_stubs` , `_stub_helper` | 动态链接         |
| `_objc_methodname`        | 方法名称         |
| `_objc_classname`         | 类名称           |
| `_objc_methtype`          | 方法类型 ( v@: ) |
| `_cstring`                | 静态字符串常量   |

### _DATA 段

`_DATA` 在内存中是紧跟在 `_TEXT` 段之后的.

| 名称                                    | 内容           |
| --------------------------------------- | -------------- |
| `_got` : Non-Lazy Symbol Pointers       | 非懒加载符号表 |
| `_la_symbol_ptr` : Lazy Symbol Pointers | 懒加载符号表   |
| `_objc_classlist`                       | 类列表         |

...

以及以一些数据源 就不一一列举了 .

### 补充

另外有一点值得提一下的就是系统库的方法 , 由于是公用的 , 存放在共享缓存中 , 那么我们的 `Mach-O` 中调用系统方法 ,

例如 : 调用 `NSLog("%@,@"haha");`

这个方法的实现肯定不在我们的 `Mach-O` 里 , 那么它如何找到方法实现呢 ?

> **其实就是 `dyld` 在进行链接的时候 , 会将 `Mach-O` 里调用存放在共享缓存中的方法进行符号绑定** , 而这个符号在 `release` 的时候是会被自动去掉的. 这也是我们经常使用收集 `bug` 工具时需要恢复符号表的原因. 而因此 `fishhook 在 `hook` 系统函数的时候名字叫` `reBind` 的原因 .

关于符号绑定这一点我们在讲 `fishhook` 的时候会详细讲述一下 .

至此 , 整个 `Mach-O` 文件结构我们已经讲述完了 . 后续在逆向的过程中涉及到具体存储内容我们会继续介绍 .

