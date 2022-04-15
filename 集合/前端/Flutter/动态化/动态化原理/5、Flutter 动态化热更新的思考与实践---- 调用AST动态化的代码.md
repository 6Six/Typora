# [Flutter 动态化热更新的思考与实践（五）---- 调用AST动态化的代码](https://juejin.cn/post/6855129008083271694)



[Flutter动态化热更新的思考与实践](https://juejin.cn/post/6844904116985004046)

[Flutter 动态化热更新的思考与实践（二）----Dart 代码转换AST](https://juejin.cn/post/6844904121300959246)

[Flutter 动态化热更新的思考与实践（三）---- 解析AST之Runtime](https://juejin.cn/post/6844904134970179592)

[Flutter 动态化热更新的思考与实践（四）---- 解析AST之Widget](https://juejin.cn/post/6844904135251197966)

> 本文主要讨论如何调用另一个AST动态化的代码

## 1. 问题

假如现在有两个自定义Widget， **A** 和 **B**， 两个Widget 都需要做动态化处理，同时 **B** 包含在 **A** 中作为Widget 树中的一个子Widget，那么在转换成AST后，如何从A Widget 中解析**B** Widget？类似的例子还有，自定义一个class **C** 需要动态化处理，同时在**A** Widget 中调用 **C** 的方法，那么如何解析 **C** 并执行其中的方法？

## 2. 解决思路

首先可以从AST的结构上做些修改，通过设计一个AST Node 的结构，来定义一个转换AST后的代码，然后在解析的时候，根据该结构的信息，获取AST完整数据，丢到我们前文中提到的[Runtime](https://juejin.cn/post/6844904134970179592)中执行即可。



![call_ast](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2020/7/30/1739d639bc4e85d7~tplv-t2oaga2asx-watermark.awebp)



### 2.1 设计 AST Node 结构

根据前面的思路，先定义一个AST Node 的数据格式：

```
{
	"type": "AstClass",
	"classId": "5942760a663c195888a64e29476ac103",
	"name": "DemoWidget",
	"version": "9fc96a1bfc60567bab9f7038175919af"
}
复制代码
```

主要说明一下"classId" 和 "version" 两个字段：

- classId 标识一份AST动态化代码，算法是取`md5({代码文件路径} + {类名称})`，基本可以唯一标识一个项目里的代码类。
- version 表示代码的版本，算法是取`md5({代码类中的内容})`，每当代码类中有变化时，version都会改变，主要在版本回滚中使用。

### 2.2 生成 AstClass Node

我们先以前面的问题作为例子，说明如何生成AST将 **A** 和 **B** 关联起来。

前文描述的问题场景是 **A** 中使用 **B**， 那么我们先生成 **B** 的AstClass Node， 思路是在`AstVisitor`中方法`visitClassDeclaration`读取`class`内容，并缓存处理后的数据，下一步需要用到：

```
@override
Map visitClassDeclaration(ClassDeclaration node) {
    var name = node.name.name;
    var source = node.toSource();
    var version = hex.encode(md5.convert(Utf8Encoder().convert(source)).bytes);
    var classId = hex.encode(
        md5.convert(Utf8Encoder().convert(filePath + '/' + name)).bytes);

    return {
      "name": name,
      "path": filePath,
      'version': version,
      'classId': classId,
    };
}
复制代码
```

其中"filePath" 字段是由外部传入，存储当前文件的路径。

要将A和B关联，还需要在模板代码中做下处理，如：

```
///关联模板 AstClass 对象
Widget selectWidget<T extends Widget>(T w) => w;

///A:
...
B b;
selectWidget<B>(b),
...
复制代码
```

然后在生成 **A** 的AST过程需要在`AstVisitor`中方法`visitMethodInvocation`识别"selectWidget"关键字，取出范型的名称，然后在代码中的"import" 声明路径里查找符合上一步缓存**B** 的AstClass Node 信息，部分代码如下：

```
///import 依赖路径
List<String> _importPathes = [];

///获取当前代码中import 的path
@override
Map visitImportDirective(ImportDirective node) {
  if (_isDebug) {
    stdout.writeln("visitImportDirective => ${node.toString()}");
  }
  var importPath = '';
  var uri = Uri.parse(node.uri.stringValue);
  if (uri.scheme == 'package') {
    var projectName = uri.pathSegments[0];
    if (projectName == _projectName) {
      for (var i = 1; i < uri.pathSegments.length; i++) {
        importPath += uri.pathSegments[i] + '/';
      }
      _importPathes.add(importPath);
    }
  } else {
    importPath = uri.path;
    var currentPath = _filePath.substring(0, _filePath.lastIndexOf('/') + 1);
    _importPathes.add(currentPath + importPath);
  }
  return null;
}

@override
Map visitMethodInvocation(MethodInvocation node) {
  Map callee;

  if (node.methodName.name == 'selectWidget') {
    //查找关联ast class
    if (node.typeArguments?.arguments?.isNotEmpty == true) {
      var astClassName =
        _safelyVisitNode(node.typeArguments.arguments[0])['name'];
      //通过import 遍历寻找ast class 文件
      for (var p in _importPathes) {
        var f1 = path.relative(p);
        for (var node in _astClassNodes) {
          var f2 = path.relative(node.filePath);
          if (f1 == f2 && astClassName == node.name) {
            //找到项目中匹配的 ast class 记录，构建AstClass Node
            callee = {
              'type': 'AstClass',
              'classId': annotation.classId,
              'version': annotation.version,
              'name': annotation.name,
            };
          }
        }
      }
    } else {
      throw "Error: selectWidget 需要定义范型";
    }

  }
  return {
        "type": "MethodInvocation",
        "callee": callee,
        "typeArguments": _safelyVisitNode(node.typeArguments),
        "argumentList":  _safelyVisitNode(node.argumentList)
       };
}
复制代码
```

### 2.3存储AstClass 信息

每次遍历代码文件生成的AST 数据需要进行存储，通过classId字段关联起来。存储方式可以是网络服务器、或本地文件、或本地数据库（如 Sqlite）等，不再详细叙述。

### 2.4解析AstClass Node

解析的思路也很简单，前文中将AstClass Node 嵌入到"MethodInvocation"节点中，在解析"MethodInvocation"节点时，取出"callee"字段，判断是否"type"是否"AstClass"，若是，取出"classId"和"version"字段，根据这两个字段从AST存储数据源中取出完整的AST数据。

## 3. 总结

这个问题的解决思路还是比较简单的，代码没什么技术含量，细节的代码就不贴了。至此，整个Flutter动态化的流程和技术原理基本都清晰明了，没有什么明显的技术障碍，该方案也具有可行性，剩下的工作就是完善细节上的工作，使其达到可以用于生产环境中。

这篇文章距离上一篇过去了几个月，其实这几个月除了处理公司的事情外，主要还是完善这个方案的细节，到目前我们已经实现了一个比较完善的版本，包括**编写模板代码** -> **生成AST** -> **上传服务器** -> **App 同步AST内容动态更新** -> **性能监控、异常监控** 整个流程。同时我们也实现了一个公司内部使用的App可视化编辑工具，给**产品**和**UI设计师**使用，**产品**或**UI设计师**通过此工具可以将编辑好的页面自动生成模板代码发送给研发，研发对接数据接口后即可直接生成AST并发布到服务器上，省去编写一大部分模板代码的工作。该工具目前还处于Beta阶段，待成熟稳定后给大家分享实现的过程和原理。



