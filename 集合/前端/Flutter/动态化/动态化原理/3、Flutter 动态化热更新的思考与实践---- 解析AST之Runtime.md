# [Flutter 动态化热更新的思考与实践（三）---- 解析AST之Runtime](https://juejin.cn/post/6844904134970179592)



[Flutter动态化热更新的思考与实践](https://juejin.cn/post/6844904116985004046)

[Flutter 动态化热更新的思考与实践（二）----Dart 代码转换AST](https://juejin.cn/post/6844904121300959246)

[Flutter 动态化热更新的思考与实践（三）---- 解析AST之Runtime](https://juejin.cn/post/6844904134970179592)

[Flutter 动态化热更新的思考与实践（四）---- 解析AST之Widget](https://juejin.cn/post/6844904135251197966)

[Flutter 动态化热更新的思考与实践（五）---- 调用AST动态化的代码](https://juejin.cn/post/6855129008083271694)

> 上一篇文章[《Flutter 动态化热更新的思考与实践（二）----Dart 代码转换AST》](https://link.juejin.cn?target=)中实现了将Dart代码转换为AST，在本篇文章将探索如何解析生成的AST数据

## 1. 何为Runtime

这里我们定义的Runtime是一个动态运行AST的容器，这要从AST解析方式说起。在开篇文章[《Flutter 动态化热更新的思考与实践》](https://link.juejin.cn?target=)提到过我们实现的这个动态化方案同样遵循MVVM思想，将UI和业务解耦。那么对AST的解析就分两部分，一个是对UI类AST的解析，一个是对业务类AST的解析。UI的解析思路很简单，我们根据AST中描述的UI结构和属性，构造出一个Widget，然后调用`setState`方法重新渲染就好，这是因为Flutter声明式编程框架的特性支持这样的操作。但是对于业务类AST就做不到这一点了，我们在App运行过程中无法将AST编译成二进制代码再执行，所以一个可行的思路就是动态解析，也是JIT的思路，边解析边运行，通过解析AST中的数据节点，映射到Dart相应的语法操作，进而执行对应的Dart代码。

## 2. 实现Runtime

要实现Runtime首先需要处理一个重要的问题，就是**变量**，代码中不可避免的要出现很多变量，来进行数据的存储和交换，语言中的语法规则也无非是对这些变量进行各种操作，所以如何在Runtime中组织**变量**是首要解决的问题。

我们知道程序运行时的变量存储在两个地方，一个是栈，一个是堆，我们再参照汇编语言操作变量的方式，引入了一个变量栈的概念。Runtime解析过程中遇到的所有变量都压入这个变量栈，比如在执行一段代码块的时候，先将代码块中的变量压栈，代码块执行完成后，再把变量出栈。我们使用变量的时候从栈顶开始向下查找，找到匹配的变量名时，就取出变量值同时停止查找，这也同样解决了变量作用域的问题。

变量的问题解决了，接下来的思路就比较简单了，不断的解析AST Node就好，流程如图：



![FlutterHot Runtime](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2020/4/22/171a15a4f01d37c2~tplv-t2oaga2asx-watermark.awebp)



## 3. 代码实现

我们使用上一篇文章中的示例代码，看一下Runtime是如何在代码中实现的：

```
//demo_blog_code.dart
int incTen(int a) {
  int b = a + 10;
  return b;
}
复制代码
```

我们分析上面这段示例代码，包含几个语法要素：

- 函数
- 函数参数
- 变量声明
- 变量赋值
- 加法表达式
- 函数返回值

将其对应到Ast 数据节点，我们需要处理的节点有：

- FunctionDeclaration
- FunctionExpression
- SimpleFormalParameter
- BlockStatement
- VariableDeclarationList
- AssignmentExpression
- BinaryExpression
- ReturnStatement

所有节点处理的代码就不贴了，贴一个代表性的`BinaryExpression`的代码实现：

```
class BinaryExpression extends AstNode {
  ///运算符
  /// * +
  /// * -
  /// * *
  /// * /
  /// * <
  /// * >
  /// * <=
  /// * >=
  /// * ==
  /// * &&
  /// * ||
  /// * %
  /// * <<
  /// * |
  /// * &
  /// * >>
  ///
  String operator;

  ///左操作表达式
  Expression left;

  ///右操作表达式
  Expression right;

  BinaryExpression(this.operator, this.left, this.right, {Map ast})
      : super(ast: ast);

  factory BinaryExpression.fromAst(Map ast) {
    if (ast != null &&
        ast['type'] == astNodeNameValue(AstNodeName.BinaryExpression)) {
      return BinaryExpression(ast['operator'], Expression.fromAst(ast['left']),
          Expression.fromAst(ast['right']),
          ast: ast);
    }
    return null;
  }

  @override
  Map toAst() => _ast;
}

复制代码
```

完整的代码见文章末的Git地址。`BinaryExpression`主要定义运算表达式语法，比如四则运算、逻辑运算等。Ast数据节点定义好后，还需要一个方法解析定义的Ast节点，同样拿`BinaryExpression`举例：

```
dynamic _executeBinaryExpression(
    BinaryExpression binaryExpression, AstVariableStack variableStack) {
  //获取左操作符的值
  var leftValue = _executeBaseExpression(binaryExpression.left, variableStack);

  //获取右操作符的值
  var rightValue =
      _executeBaseExpression(binaryExpression.right, variableStack);

  //操作符
  switch (binaryExpression.operator) {
    case '+':
      return leftValue + rightValue;
    case '-':
      return leftValue - rightValue;
    case '*':
      return leftValue * rightValue;
    case '/':
      return leftValue / rightValue;
    case '<':
      return leftValue < rightValue;
    case '>':
      return leftValue > rightValue;
    case '<=':
      return leftValue <= rightValue;
    case '>=':
      return leftValue >= rightValue;
    case '==':
      return leftValue == rightValue;
    case '&&':
      return leftValue && rightValue;
    case '||':
      return leftValue || rightValue;
    case '%':
      return leftValue % rightValue;
    case '<<':
      return leftValue << rightValue;
    case '|':
      return leftValue | rightValue;
    case '&':
      return leftValue & rightValue;
    case '>>':
      return leftValue >> rightValue;
    default:
      return null;
  }
}

复制代码
```

方法会返回动态执行运算表达式`BinaryExpression`的运算结果，其他Ast节点的解析也是同样的原理。

最后再来看看Runtime的代码实现：

```
class AstRuntime {
  ///Ast 类定义
  AstClass _astClass;
  ///变量栈
  AstVariableStack _variableStack;

  AstRuntime(Map ast) {
    if (ast['type'] == astNodeNameValue(AstNodeName.Program)) {
      var body = ast['body'] as List;
      _variableStack = AstVariableStack();
      _variableStack.blockIn();
      body?.forEach((b) {
        if (b['type'] == astNodeNameValue(AstNodeName.ClassDeclaration)) {
          //解析类
          _astClass = AstClass.fromAst(b, variableStack: _variableStack);
        } else if (b['type'] ==
            astNodeNameValue(AstNodeName.FunctionDeclaration)) {
          //解析全局函数
          var func = AstFunction.fromAst(b);
          _variableStack.setFunctionInstance<AstFunction>(func.name, func);
        }
      });
    }
  }

  ///调用类方法，注意参数列表顺序与模版代码中相同
  Future callMethod(String methodName, {List params}) async {
    if (_astClass != null) {
      return _astClass.callMethod(methodName, params: params);
    }
    return Future.value();
  }

  ///调用全局函数，注意参数列表顺序与模版代码中相同
  Future callFunction(String functionName, {List params}) async {
    var function =
        _variableStack.getFunctionInstance<AstFunction>(functionName);
    if (function != null) {
      return function.invoke(params, variableStack: _variableStack);
    }
    return Future.value();
  }
}
复制代码
```

我们在Command-line程序中测试下Runtime：

```
main(List<String> arguments) async {
  exitCode = 0; // presume success
  final parser = ArgParser()..addFlag("file", negatable: false, abbr: 'f');

  var argResults = parser.parse(arguments);
  final paths = argResults.rest;
  if (paths.isEmpty) {
    stdout.writeln('No file found');
  } else {
    var ast = await generate(paths[0]);
    var astRuntime = AstRuntime(ast);
    var res = await astRuntime.callFunction('incTen', params: [100]);
    stdout.writeln('Invoke incTec(100) result: $res');
  }
}
复制代码
```

我们执行上面的测试代码：



![image-20200422174629833](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2020/4/22/171a158eb3ce0a48~tplv-t2oaga2asx-watermark.awebp)



输出"110"， 符合源代码的计算逻辑。

完整的代码实现详见：[DynamicFlutter](https://link.juejin.cn?target=https%3A%2F%2Fgithub.com%2FNewcore-mobile%2FDynamicFlutter)

