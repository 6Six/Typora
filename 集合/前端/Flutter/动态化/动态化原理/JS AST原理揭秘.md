# [JS AST原理揭秘](https://zhaomenghuan.js.org/blog/js-ast-principle-reveals.html)



## 前言

抽象语法树 AST 一直都想系统性的学习整理一下，这篇文章将以实际的例子讲解一下 AST 的基本内容。之前在做基于 Flutter 的类似小程序语法的开发框架 [flutter-mini-program](https://github.com/zhaomenghuan/flutter-mini-program)时，通过 Dart 解析 HTML 标签和 CSS 标签实现了 HTML + CSS 对 Flutter 视图层的描述；对于逻辑层的设计，Flutter 不支持动态加载 Dart 文件，也不支持 JS 语法运行环境，当时就想过一种思路，基于 Dart 解析 JS 语法的方式是否行得通，Dart 官方具备 Dart2js 框架，Dart SDK 里面具备 JS 和 JS AST 相关的包，原则上这个思路可行，但是没时间就搁置了这个想法。后来看到闲鱼技术团队提出了一种基于 [Flutter Dart AST 实现热更新](https://www.yuque.com/xytech/flutter/emdguh) 方式，目前看也是一种不错的方式。最近在做小程序一套代码转换成多套代码的框架，看到主流小程序框架里面有基于 AST 转换的逻辑，所以本文尝试重新梳理一下 JS AST 的内容。

## 什么是 AST ?

常见编译型语言（例如：Java）编译程序一般步骤分为：词法分析->语法分析->语义检查->代码优化和字节码生成。具体的编译流程如下图：

![img](https://zhaomenghuan.js.org/assets/img/common-lang-compile.8843ac98.png)

对于解释型语言（例如 JavaScript）来说，通过词法分析 -> 语法分析 -> 语法树，就可以开始解释执行了。

### 词法分析/分词(Tokenizing/Lexing)

将字符流(char stream)转换为记号流(token stream)，由字符串组成的字符分解成有意义的代码块，这些代码块称为词法单元。例如：一段 JS 代码 `var name = 'Hello Flutter';` 会被分解为词法单元：`var`、`name`、`=`、`Hello Flutter`、`;`。

```json
[
  {
    "type": "Keyword",
    "value": "var"
  },
  {
    "type": "Identifier",
    "value": "name"
  },
  {
    "type": "Punctuator",
    "value": "="
  },
  {
    "type": "String",
    "value": "'Hello Flutter'"
  },
  {
    "type": "Punctuator",
    "value": ";"
  }
]
```

最小词法单元主要有空格、注释、字符串、数字、标志符、运算符、括号等。

### 语法分析/解析(Parsing)

将词法单元流转换成一个由元素逐级嵌套所组成的代表了程序语法结构的树。这个树称为 “抽象语法树” AST（Abstract Syntax Tree）。

词法分析和语法分析不是完全独立的，而是交错进行的，也就是说，词法分析器不会在读取所有的词法记号后再使用语法分析器来处理。在通常情况下，每取得一个词法记号，就将其送入语法分析器进行分析。

![img](https://zhaomenghuan.js.org/assets/img/js-token-ast.f635f3cf.png)

语法分析的过程就是把词法分析所产生的记号生成语法树，通俗地说，就是把从程序中收集的信息存储到数据结构中。注意，在编译中用到的数据结构有两种：符号表和语法树。

- **符号表**：就是在程序中用来存储所有符号的一个表，包括所有的字符串变量、直接量字符串，以及函数和类。
- **语法树**：就是程序结构的一个树形表示，用来生成中间代码。

如果 JavaScript 解释器在构造语法树的时候发现无法构造，就会报语法错误，并结束整个代码块的解析。对于传统强类型语言来说，在通过语法分析构造出语法树后，翻译出来的句子可能还会有模糊不清的地方，需要进一步的语义检查。语义检查的主要部分是类型检查。例如，函数的实参和形参类型是否匹配。但是，对于弱类型语言来说，就没有这一步。

经过编译阶段的准备，JavaScript 代码在内存中已经被构建为语法树，然后 JavaScript 引擎就会根据这个语法树结构边解释边执行。

我们可以使用在线工具生成 AST : http://esprima.org/ 或 https://astexplorer.net/

`var name = 'Hello Flutter';` 转成 AST 如下：

```json
{
  "type": "Program",
  "body": [
    {
      "type": "VariableDeclaration",
      "declarations": [
        {
          "type": "VariableDeclarator",
          "id": {
            "type": "Identifier",
            "name": "name"
          },
          "init": {
            "type": "Literal",
            "value": "Hello Flutter",
            "raw": "'Hello Flutter'"
          }
        }
      ],
      "kind": "var"
    }
  ],
  "sourceType": "script"
}
```

> 注意：出于简化的目的移除了某些属性

这样的每一层结构也被叫做 节点（Node）。 一个 AST 可以由单一的节点或是成百上千个节点构成。 它们组合在一起可以描述用于静态分析的程序语法。

### 代码生成(Code Generation)

将 AST 转换为可执行代码的过程称被称为代码生成。代码生成步骤把最终（经过一系列转换之后）的 AST 转换成字符串形式的代码，同时还会创建[源码映射（source maps）](https://www.html5rocks.com/en/tutorials/developertools/sourcemaps)。代码生成其实很简单：深度优先遍历整个 AST，然后构建可以表示转换后代码的字符串。

### ESTree AST Node

ESTree AST 节点表示为 Node 对象，它可以具有任何原型继承但实现以下接口：

```js
interface Node {
  type: string;
}
```

字符串形式的 type 字段表示节点的类型（如： "FunctionDeclaration"，"Identifier"，或 "BinaryExpression"）。每一种类型的节点定义了一些附加属性用来进一步描述该节点类型。

```js
{
  type: ...,
  start: 0,
  end: 38,
  loc: {
    start: {
      line: 1,
      column: 0
    },
    end: {
      line: 3,
      column: 1
    }
  },
  ...
}
```

每一个节点都会有 start，end，loc 这几个属性。

#### Program 节点

```js
interface Program <: Node {
  type: "Program";
  body: [ Directive | Statement ];
}
```

Program 是一个完整的程序源代码树。

#### Declaration 节点

```js
interface Declaration <: Statement { }
```

具体可以参考：[estree](https://github.com/estree/estree/blob/master/es5.md)

### JS AST 工具

JS 生态基于 AST 实现的一些工具有很多，例如：

- babel: 实现 JS 编译，转换过程是 AST 的转换
- ESlint: 代码错误或风格的检查，发现一些潜在的错误
- IDE 的错误提示、格式化、高亮、自动补全等
- UglifyJS 压缩代码
- 代码打包工具 webpack

可以参考的 AST 生成及代码合成的项目：

- [acorn](https://github.com/acornjs/acorn)
- [babel-core](https://github.com/babel/babel/tree/master/packages/babel-core)
- [the-super-tiny-compiler](https://github.com/jamiebuilds/the-super-tiny-compiler)
- [LangSandbox](https://github.com/ftomassetti/LangSandbox)

## Babel 的工作原理

Babel 是一个通用的多功能的 JavaScript 编译器，更确切地说是源码到源码的编译器，通常也叫做"转换编译器"(transpiler)。很多浏览器目前还不支持 ES6 的代码，但是我们可以通过 Babel 将 ES6 的代码转译成 ES5 代码，让所有的浏览器都能理解的代码，这就是 Babel 的作用。

Babel 的编译过程和大多数其他语言的编译器大致相同，可以分为三个阶段。

1. 解析(parse)：将代码字符串解析成抽象语法树。
2. 转换(transform)：对抽象语法树进行转换操作。
3. 生成(generate): 根据变换后的抽象语法树再生成代码字符串。

![img](https://zhaomenghuan.js.org/assets/img/babel-function.42267193.png)

### [@babel/parser](https://babeljs.io/docs/en/next/babel-parser)

@babel/parser 将源代码解析成 AST。

@babel/parser 是 Babel 的 JavaScript 解析器，原名 Babylon。最初是 从 Acorn 项目 fork 出来的。Acorn 非常快，易于使用，并且针对非标准特性(以及那些未来的标准特性)设计了一个基于插件的架构。

安装：

```text
$ npm install --save-dev @babel/parser
```

先从解析一个代码字符串开始：

```js
const babelParser = require("@babel/parser");

const code = `let square = (n) => {
  return n * n;
}`;

let ast = babelParser.parse(code);
console.log(ast);
```

> Babel 使用一个基于 ESTree 并修改过的 AST，它的内核说明文档可以在 [ast-spec](https://github.com/babel/babel/blob/master/doc/ast/spec.md) 查看。

### [@babel/generator](https://babeljs.io/docs/en/next/babel-generator)

@babel/generator 将 AST 解码生 JS 代码

```js
const babelParser = require("@babel/parser");
const generate = require("@babel/generator");

const astA = babelParser.parse("var a = 1;");
const astB = babelParser.parse("var b = 2;");
const ast = {
  type: "Program",
  body: [].concat(astA.program.body, astB.program.body)
};

const { code } = generate.default(ast);
console.log(code);
```

### [@babel/plugin-transform-?](https://github.com/babel/babel/tree/master/packages)

Babel 官方提供了一系列用于代码转换的插件，例如 `@babel/plugin-transform-typescript`，我们通常会通过 babel 的插件来完成代码转换。

### [@babel/core](https://babeljs.io/docs/en/next/babel-core)

包括了整个 Babel 工作流，也就是说在 `@babel/core` 里面我们会使用到 `@babel/parser`、`transformer[s]`、以及`@babel/generator`