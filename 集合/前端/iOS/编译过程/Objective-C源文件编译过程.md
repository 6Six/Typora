# [Objective-C源文件编译过程](https://cloud.tencent.com/developer/article/1520878)



## 简介

Objective-C文件的编译过程主要包括clang前端的预处理、编译、后端优化中间表示、生成汇编指令、链接、生成机器码这几个步骤。我们可以借助`clang -ccc-print-phases xxx.m`命令查看某个OC源文件的编译的过程，如下： 输入命令

```javascript
clang -ccc-print-phases main.m
```

命令行输出

```javascript
0: input, "main.m", objective-c
1: preprocessor, {0}, objective-c-cpp-output
2: compiler, {1}, ir
3: backend, {2}, assembler
4: assembler, {3}, object
5: linker, {4}, image
6: bind-arch, "x86_64", {5}, image
```

可见编译过程先后是读取源文件、预处理、编译生成中间表示、生成汇编码、链接生成image、绑定架构生成对应机器码。本篇文章我们着重分析预处理、编译、生成汇编代码、链接这4个步骤。

## 预处理

通常，一个源程序可能被分割为多个模块，并存放于独立的文件中，把源程序“聚合”在一起的任务叫做预处理。预处理操作由预处理器独立完成。正如我们所知，预处理器通常把那些称为宏的缩写形式转换为源语言的语句。比如宏定义、条件编译、文件包含。 如下命令可以对.c、.m源文件进行预处理，其中参数`-E`就是对源文件进行预处理操作：

```javascript
clang -E xxx.m
```

如果我们的.m文件中import（文件包含）了其他的文件或者其他的库，执行以上命令对OC源文件进行预处理可能会遇到如下错误：

```javascript
main.m:9:9: fatal error: 'UIKit/UIKit.h' file not found
#import <UIKit/UIKit.h>
^~~~~~~~~~~~~~~
# 10 "main.m"
# 1 "./AppDelegate.h" 1
# 11 "./AppDelegate.h"
```

解决办法： 添加-isysroot参数来指定iPhoneSimulator.sdk的路径`-isysroot /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneSimulator.platform/Developer/SDKs/iPhoneSimulator.sdk` 预处理 -E

```javascript
clang -x objective-c -E -isysroot /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneSimulator.platform/Developer/SDKs/iPhoneSimulator.sdk main.m
```

当然，把-E换成 -rewrite-objc就可以把OC源文件转成C++文件，如下： OC转C++ -rewrite-objc

```javascript
clang -x objective-c -rewrite-objc -isysroot /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneSimulator.platform/Developer/SDKs/iPhoneSimulator.sdk main.m
```

除了使用Clang查看OC源文件的预处理过程，还可以使用`xcrun`命令，如下： 注意下面的命令中必须加上-E（-E代表对文件进行预处理）

```javascript
xcrun -sdk iphoneos clang -arch armv7 -F Foundation -E -fobjc-arc -c main.m
```

如果要生成.o目标文件，只需要在最后指定目标文件的名称，如下：

```javascript
xcrun -sdk iphoneos clang -arch armv7 -F Foundation -E -fobjc-arc -c main.m -o main.o
```

## 编译

### 词法分析

编译器中负责将程序分解为一个一个符号的部分，一般称为“词法分析器”（引用自《C Traps and Pitfalls》）。 词法分析器读入组成源程序的字符流，并且将他们组织成为有意义的词素(lexeme)序列。对于每个词素，词法分析器产生词法单元`token（符号）`作为输出（引用自《编译原理》）。 token指的是程序的一个基本组成单元—词法单元。token的作用相当于一个句子中的单词，从某种意义上来说，一个单词无论出现在哪个句子中，它代表的意思都是一样的，是一个表义的基本单元。与此类似，token就是程序中的一个基本信息单元。词法分析器将源文件的字符流转换为token的过程被称作**词法分析（lexical anaysis）**。

对某一个源文件进行词法分析，可以使用下面这个命令

```javascript
clang -fmodules -E -Xclang -dump-tokens main.m
```

当然，和预处理一样，如果源文件中有import其他文件，那么还需要使用-isysroot 参数来指定iPhoneSimulator.sdk的路径，如下：

```javascript
clang -fmodules -E -Xclang -dump-tokens -isysroot /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneSimulator.platform/Developer/SDKs/iPhoneSimulator.sdk main.m
```

本文本着示例简单、避免干扰的原则，将会对如下源码进行词法分析

```javascript
//
//  main.m
//  ArrayWithArray:
//
//  Created by sw on 2018/1/5.
//  Copyright © 2018年 sw. All rights reserved.
//

#include <stdio.h>
#define name "ws"

int main(int argc, char * argv[]) {
@autoreleasepool {
printf("hello %s", name);
return 1;
}
}
```

如下是main.m文件词法分析后的结果

```javascript
annot_module_include '#include <stdio.h>
#define name "ws"

int main(int argc, char * argv[]) {
@autoreleasepool {
printf("hello %s", name);
'        Loc=<main.m:9:1>
int 'int'     [StartOfLine]    Loc=<main.m:12:1>
identifier 'main'     [LeadingSpace]    Loc=<main.m:12:5>
l_paren '('        Loc=<main.m:12:9>
int 'int'        Loc=<main.m:12:10>
identifier 'argc'     [LeadingSpace]    Loc=<main.m:12:14>
comma ','        Loc=<main.m:12:18>
char 'char'     [LeadingSpace]    Loc=<main.m:12:20>
star '*'     [LeadingSpace]    Loc=<main.m:12:25>
identifier 'argv'     [LeadingSpace]    Loc=<main.m:12:27>
l_square '['        Loc=<main.m:12:31>
r_square ']'        Loc=<main.m:12:32>
r_paren ')'        Loc=<main.m:12:33>
l_brace '{'     [LeadingSpace]    Loc=<main.m:12:35>
at '@'     [StartOfLine] [LeadingSpace]    Loc=<main.m:13:5>
identifier 'autoreleasepool'        Loc=<main.m:13:6>
l_brace '{'     [LeadingSpace]    Loc=<main.m:13:22>
identifier 'printf'     [StartOfLine] [LeadingSpace]    Loc=<main.m:14:9>
l_paren '('        Loc=<main.m:14:15>
string_literal '"hello %s"'        Loc=<main.m:14:16>
comma ','        Loc=<main.m:14:26>
string_literal '"ws"'     [LeadingSpace]    Loc=<main.m:14:28 <Spelling=main.m:10:14>>
r_paren ')'        Loc=<main.m:14:32>
semi ';'        Loc=<main.m:14:33>
return 'return'     [StartOfLine] [LeadingSpace]    Loc=<main.m:15:9>
numeric_constant '1'     [LeadingSpace]    Loc=<main.m:15:16>
semi ';'        Loc=<main.m:15:17>
r_brace '}'     [StartOfLine] [LeadingSpace]    Loc=<main.m:16:5>
r_brace '}'     [StartOfLine]    Loc=<main.m:17:1>
eof ''        Loc=<main.m:17:2>
```

如上，**int 'int'     [StartOfLine]    Loc=<main.m:12:1>**就是一个token，**identifier 'main'     [LeadingSpace]    Loc=<main.m:12:5>**也是一个token。每个token后面的Loc代表这个token在源文件中的位置。例如Loc=<main.m:12:1>代表这个token位于main.m文件中的第12行第1个位置。注意：这里的位置是从1开始，而非0。 而上述token中的'int'、'main' 就是**词素**。 ps：由上面词法分析后的结果和源文件对照可知，注释虽然没有真实的意义，但是注释占用的行依旧是有效的，在词法分析阶段并没有被忽略掉。这样的好处是，词法分析后的token的Loc就是真实的Location。

### 语法分析

将词法分析的token解析处理成抽象语法树AST（abstract syntax tree）的过程称作**语法分析（semantic analysis）**。即语法分析的输入是token，输出是AST。AST则更加直观的反映了代码的内部结构和逻辑。使用`clang -fmodules -fsyntax-only -Xclang -ast-dump main.m`可以对源文件进行语法分析，如下：

```javascript
TranslationUnitDecl 0x7f8c10009ee8 <<invalid sloc>> <invalid sloc>
|-TypedefDecl 0x7f8c1000a780 <<invalid sloc>> <invalid sloc> implicit __int128_t '__int128'
| `-BuiltinType 0x7f8c1000a480 '__int128'
|-TypedefDecl 0x7f8c1000a7e8 <<invalid sloc>> <invalid sloc> implicit __uint128_t 'unsigned __int128'
| `-BuiltinType 0x7f8c1000a4a0 'unsigned __int128'
|-TypedefDecl 0x7f8c1000a880 <<invalid sloc>> <invalid sloc> implicit SEL 'SEL *'
| `-PointerType 0x7f8c1000a840 'SEL *'
|   `-BuiltinType 0x7f8c1000a6e0 'SEL'
|-TypedefDecl 0x7f8c1000a958 <<invalid sloc>> <invalid sloc> implicit id 'id'
| `-ObjCObjectPointerType 0x7f8c1000a900 'id'
|   `-ObjCObjectType 0x7f8c1000a8d0 'id'
|-TypedefDecl 0x7f8c1000aa38 <<invalid sloc>> <invalid sloc> implicit Class 'Class'
| `-ObjCObjectPointerType 0x7f8c1000a9e0 'Class'
|   `-ObjCObjectType 0x7f8c1000a9b0 'Class'
|-ObjCInterfaceDecl 0x7f8c1000aa88 <<invalid sloc>> <invalid sloc> implicit Protocol
|-TypedefDecl 0x7f8c100275e8 <<invalid sloc>> <invalid sloc> implicit __NSConstantString 'struct __NSConstantString_tag'
| `-RecordType 0x7f8c10027400 'struct __NSConstantString_tag'
|   `-Record 0x7f8c1000ab50 '__NSConstantString_tag'
|-TypedefDecl 0x7f8c10027680 <<invalid sloc>> <invalid sloc> implicit __builtin_ms_va_list 'char *'
| `-PointerType 0x7f8c10027640 'char *'
|   `-BuiltinType 0x7f8c10009f80 'char'
|-TypedefDecl 0x7f8c10027948 <<invalid sloc>> <invalid sloc> implicit __builtin_va_list 'struct __va_list_tag [1]'
| `-ConstantArrayType 0x7f8c100278f0 'struct __va_list_tag [1]' 1
|   `-RecordType 0x7f8c10027770 'struct __va_list_tag'
|     `-Record 0x7f8c100276d0 '__va_list_tag'
|-ImportDecl 0x7f8c100281c0 <main.m:9:1> col:1 implicit Darwin.C.stdio
|-FunctionDecl 0x7f8c100f6a78 <line:12:1, line:17:1> line:12:5 main 'int (int, char **)'
| |-ParmVarDecl 0x7f8c10028210 <col:10, col:14> col:14 argc 'int'
| |-ParmVarDecl 0x7f8c10028320 <col:20, col:32> col:27 argv 'char **':'char **'
| `-CompoundStmt 0x7f8c100f7188 <col:35, line:17:1>
|   `-ObjCAutoreleasePoolStmt 0x7f8c100f7178 <line:13:5, line:16:5>
|     `-CompoundStmt 0x7f8c100f7158 <line:13:22, line:16:5>
|       |-CallExpr 0x7f8c100f70a0 <line:14:9, col:32> 'int'
|       | |-ImplicitCastExpr 0x7f8c100f7088 <col:9> 'int (*)(const char *, ...)' <FunctionToPointerDecay>
|       | | `-DeclRefExpr 0x7f8c100f6f58 <col:9> 'int (const char *, ...)' Function 0x7f8c100f6b88 'printf' 'int (const char *, ...)'
|       | |-ImplicitCastExpr 0x7f8c100f70f0 <col:16> 'const char *' <BitCast>
|       | | `-ImplicitCastExpr 0x7f8c100f70d8 <col:16> 'char *' <ArrayToPointerDecay>
|       | |   `-StringLiteral 0x7f8c100f6fb8 <col:16> 'char [9]' lvalue "hello %s"
|       | `-ImplicitCastExpr 0x7f8c100f7108 <line:10:14> 'char *' <ArrayToPointerDecay>
|       |   `-StringLiteral 0x7f8c100f7028 <col:14> 'char [3]' lvalue "ws"
|       `-ReturnStmt 0x7f8c100f7140 <line:15:9, col:16>
|         `-IntegerLiteral 0x7f8c100f7120 <col:16> 'int' 1
`-<undeserialized declarations>
```

有了抽象语法树，clang就可以对这个树进行分析，找出代码中的错误，很多编译期的检查都是针对于抽象语法树的检查。比如类型不匹配，未实现对应的方法。

AST是开发者编写clang插件主要交互的数据结构，clang也提供很多API去读取AST。详情参考：[Introduction to the Clang AST](https://links.jianshu.com/go?to=https%3A%2F%2Fclang.llvm.org%2Fdocs%2FIntroductionToTheClangAST.html)。

### 语义分析

使用语法分析产生的语法树和符号表检查源程序是否和语言定义的语义一致的过程被称为**语义分析**。这个定义听起来比较绕，后面会解释。语义分析的过程同时也收集类型信息，并把类型信息存储在语法树或符号表中，以便随后的中间代码生成过程中使用。 语义分析一个重要的部分就是“类型检查”和“自动类型转换”。编译器检查每个运算符是否有匹配的运算分量。所谓运算分量就是指被运算符操作的量。拿C语言的语义分析举例，比如a + b, 其中“+”就是运算符，a和b就是这个运算符的分量。如果a和b都是整型或浮点型，这说明“+”运算符具有匹配的运算分量。如果a或b其中一个是字符串类型，则说明“+”运算符不具备匹配的运算分量。又比如，很多语言中要求数组的下标是一个非负整数，如果浮点数作为下标，编译器就必须报告错误。

### 生成中间代码

>  在把源程序翻译成目标代码的过程中，一个编译器可能构造出一个或多个中间表示（Intermediate Representation或IR）。这些中间表示可以有多种形式。语法树（AST）就是一种中间表示形式。--摘抄自《编译原理》 我们已经知道，语法分析生成AST，语义分析会对根据AST和符号表对源程序进行检查。那么语法分析和语义分析都完成后，clang会遍历AST生成一种明确的、低级的或类机器语言的中间表示。LLVM IR是LLVM套件里面的中间表示（LLVM Intermediate Representation），LLVM IR也是前端（clang）的输出，后端的输入。 LLVM IR有3种表示形式，分别是： 

- text格式：便于阅读的文本格式，类似于汇编语言，拓展名.ll， 

![img](https://ask.qcloudimg.com/http-save/yehe-2283478/3xidyfvc9i.svg)

 xcrun clang -S -emit-llvm main.c -o main.ll

- memory格式：内存格式
- bitcode格式：二进制格式，拓展名.bc， $ clang -c -emit-llvm main.m ps：以上三种形式的本质是等价的，就好比水可以有气体、液体、固体3种形态。 我们使用`clang -S -emit-llvm main.m`命令来获取text格式的文件，文件后缀名是.ll，使用文本编辑器即可打开，如下：

```javascript
; ModuleID = 'main.m'
source_filename = "main.m"
target datalayout = "e-m:o-i64:64-f80:128-n8:16:32:64-S128"
target triple = "x86_64-apple-macosx10.14.0"

@.str = private unnamed_addr constant [9 x i8] c"hello %s\00", align 1
@.str.1 = private unnamed_addr constant [3 x i8] c"ws\00", align 1

; Function Attrs: noinline optnone ssp uwtable
define i32 @main(i32, i8**) #0 {
%3 = alloca i32, align 4
%4 = alloca i32, align 4
%5 = alloca i8**, align 8
store i32 0, i32* %3, align 4
store i32 %0, i32* %4, align 4
store i8** %1, i8*** %5, align 8
%6 = call i8* @objc_autoreleasePoolPush() #2
%7 = call i32 (i8*, ...) @printf(i8* getelementptr inbounds ([9 x i8], [9 x i8]* @.str, i32 0, i32 0), i8* getelementptr inbounds ([3 x i8], [3 x i8]* @.str.1, i32 0, i32 0))
store i32 1, i32* %3, align 4
call void @objc_autoreleasePoolPop(i8* %6)
%8 = load i32, i32* %3, align 4
ret i32 %8
}

declare i8* @objc_autoreleasePoolPush()

declare i32 @printf(i8*, ...) #1

declare void @objc_autoreleasePoolPop(i8*)

attributes #0 = { noinline optnone ssp uwtable "correctly-rounded-divide-sqrt-fp-math"="false" "disable-tail-calls"="false" "less-precise-fpmad"="false" "no-frame-pointer-elim"="true" "no-frame-pointer-elim-non-leaf" "no-infs-fp-math"="false" "no-jump-tables"="false" "no-nans-fp-math"="false" "no-signed-zeros-fp-math"="false" "no-trapping-math"="false" "stack-protector-buffer-size"="8" "target-cpu"="penryn" "target-features"="+cx16,+fxsr,+mmx,+sahf,+sse,+sse2,+sse3,+sse4.1,+ssse3,+x87" "unsafe-fp-math"="false" "use-soft-float"="false" }
attributes #1 = { "correctly-rounded-divide-sqrt-fp-math"="false" "disable-tail-calls"="false" "less-precise-fpmad"="false" "no-frame-pointer-elim"="true" "no-frame-pointer-elim-non-leaf" "no-infs-fp-math"="false" "no-nans-fp-math"="false" "no-signed-zeros-fp-math"="false" "no-trapping-math"="false" "stack-protector-buffer-size"="8" "target-cpu"="penryn" "target-features"="+cx16,+fxsr,+mmx,+sahf,+sse,+sse2,+sse3,+sse4.1,+ssse3,+x87" "unsafe-fp-math"="false" "use-soft-float"="false" }
attributes #2 = { nounwind }

!llvm.module.flags = !{!0, !1, !2, !3, !4, !5, !6, !7}
!llvm.ident = !{!8}

!0 = !{i32 2, !"SDK Version", [2 x i32] [i32 10, i32 14]}
!1 = !{i32 1, !"Objective-C Version", i32 2}
!2 = !{i32 1, !"Objective-C Image Info Version", i32 0}
!3 = !{i32 1, !"Objective-C Image Info Section", !"__DATA,__objc_imageinfo,regular,no_dead_strip"}
!4 = !{i32 4, !"Objective-C Garbage Collection", i32 0}
!5 = !{i32 1, !"Objective-C Class Properties", i32 64}
!6 = !{i32 1, !"wchar_size", i32 4}
!7 = !{i32 7, !"PIC Level", i32 2}
!8 = !{!"Apple LLVM version 10.0.1 (clang-1001.0.46.4)"}
```

Clang还会收集源程序的信息，并把信息存放在符号表（symbol table）中。符号表和LLVM IR会被传递给后端。

### 代码生成

代码生成（CodeGen）由代码生成器完成。以源程序的中间表示（IR）作为输入，并把它映射到目标语言。如果目标语言是机器代码，那么就必须为程序使用的每个变量选择寄存器或内存位置。然后中间指令被翻译成为能够完成相同任务的机器指令序列。代码生成的一个至关重要的方面是合力分配寄存器以存放变量的值。

## LLVM IR

有些编译器的结构单纯的分为前端和后端，比如GCC。而LLVM的结构并不是单纯的分为前端和后端。LLVM编译器集合是围绕着一组精心设计的中间表示形式而创建的，这些中间表示形式使得我们可以把特定语言的前端和特定目标机的后端相结合。使用这些集合，我们可以把不同的前端和某个目标机的后端结合起来，为不同的源语言建立该目标机上的编译器。类似的，我们可以把一个前端和不同的目标机后端结合，简历针对不同目标机的编译器。 这样说可能比较绕，本质上是LLVM IR优化器会做一些与代码无关的优化，所以如果LLVM将来需要支持一门新的编程语言，只需针对这个编程语言提供一个新的前端。如果将来LLVM需要支持一款新的机器架构，只需要针对这款机器架构提供一个新的后端。而LLVM IR优化器是通用的。这样一来LLVM就变得易扩展。

## 生成汇编代码

LLVM对IR进行优化后，会针对不同架构生成不同的目标代码，最后以汇编代码的格式输出：

生成arm 64汇编：

```javascript
xcrun clang -S main.c -o main.s
```

## 汇编器

汇编器以汇编代码作为输入，将汇编代码转换为机器代码，最后输出目标文件(object file)。

```javascript
xcrun clang -fmodules -c main.c -o main.o
```

## 链接

链接器把编译产生的.o文件和（dylib,a,tbd）文件，生成一个mach-o文件。

```javascript
xcrun clang main.o -o main
```

我们就得到了一个mach o格式的可执行文件

```javascript
$ file main
main: Mach-O 64-bit executable x86_64
$ ./main 
hello debug
```

在用nm命令，查看可执行文件的符号表：

```javascript
$ nm -nm main
```

