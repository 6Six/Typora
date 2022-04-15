# Dart语法篇之函数的使用(四)



简述:

在上一篇文章中我们详细地研究了一下集合有关内容，包括集合的操作符的使用甚至我们还深入到源码实现原理，从原理上掌握集合的使用。那么这篇文章来研究一下Dart的另一个重要语法: **函数**。

这篇主要会涉及到: 函数命名参数、可选参数、参数默认、闭包函数、箭头函数以及函数作为对象使用。

### 一、函数参数

在Dart函数参数是一个比较重要的概念，此外它涉及到概念的种类比较多，比如位置参数、命名参数、可选位置参数、可选命名参数等等。函数总是有一个所谓形参列表，虽然这个参数列表可能为空，比如**getter函数就是没有参数列表的**. 此外在Dart中函数参数大致可分为两种: **位置参数和命名参数**，来一张图理清它们的概念关系



![img](https://pic3.zhimg.com/80/v2-4507b0f462f98194796c0cdb2f2e757a_1440w.jpg)



### 1、位置参数

**位置参数可以必需的也可以是可选**。

- 无参数

```dart
//无参数类型-这是不带函数参数或者说参数列表为空
String getDefaultErrorMsg() => 'Unknown Error!';
//无参数类型-等价于上面函数形式，同样是参数列表为空
get getDefaultErrorMsg => 'Unknown Error!';
```

- 必需位置参数

```dart
//必需位置参数类型-这里的exception是必需的位置参数
String getErrorMsg(Exception exception) => exception.toString();
```

- 可选位置参数

```dart
//注意: 可选位置参数是中括号括起来表示，例如[String error]
String getErrorMsg([String error]) => error ?? 'Unknown Error!';
```

- 必需位置参数和可选位置参数混合

```dart
//注意: 可选位置参数必须在必需位置参数的后面
String getErrorMsg(Exception exception, [String extraInfo]) => '${exception.toString()}---$extraInfo';
```

### 2、命名参数

命名参数**始终是可选参数**。为什么是命名参数，这是因为在调用函数时可以任意指定参数名来传参。

- 可选命名参数

```dart
//注意: 可选命名参数是大括号括起来表示，例如{num a, num b, num c, num d}
num add({num a, num b, num c, num d}) {
   return a + b + c + d;
}
//调用
main() {
   print(add(d: 4, b: 3, a: 2, c: 1));//这里的命名参数就是可以任意顺序指定参数名传值,例如d: 4, b: 3, a: 2, c: 1
}
```

- 必需位置参数和可选命名参数混合

```dart
//注意: 可选命名参数必须在必需位置参数的后面
num add(num a, num b, {num c, num d}) {
   return a + b + c + d;
}
//调用
main() {
   print(add(4, 5, d: 3, c: 1));//这里的命名参数就是可以任意顺序指定参数名传值,例如d: 3, c: 1,但是必需参数必须按照顺序传参。
}
```

- 注意: 可选位置参数和可选命名参数不能混合在一起使用，因为可选参数列表只能位于整个函数形参列表的最后。

```dart
void add7([num a, num b], {num c, num d}) {//非法声明，想想也没有必要两者一起混合使用场景。所以
   ...
}
```

### 3、关于可选位置参数`[num a, num b]`和可选命名参数`{num a, num b}`使用场景

可能问题来了，啥时候使用可选位置参数，啥时候使用可选命名参数呢?

这里给个建议: **首先，参数是非必需的也就是可选的，如果可选参数个数只有一个建议直接使用可选位置参数`[num a, num b]`；如果可选参数个数是多个的话建议用可选命名参数`{num a, num b}`.** 因为多个参数可选，指定参数名传参对整体代码可读性有一定的增强。

### 4、参数默认值(针对可选参数)

首先，需要明确一点，参数默认值只针对可选参数才能添加的。可以使用 **=** 来定义命名和位置参数的默认值。**默认值必须是编译时常量。如果没有提供默认值，则默认值为null**。

- 可选位置参数默认值

```dart
num add(num a, num b, num c, [num d = 5]}) {//使用=来赋值默认值
    return a + b + c + d;
}
main() {
    print(add(1, 2, 3));//有默认值参数可以省略不传 实际上求和结果是: 1 + 2 + 3 + 5(默认值)
    print(add(1, 2, 3, 4));//有默认值参数指定传入4，会覆盖默认值，所以求和结果是: 1 + 2 + 3 + 4
}
```

- 可选命名参数默认值

```dart
num add({num a, num b, num c = 3, num d = 4}) {
    return a + b + c + d;
}
main() {
    print(add(100, 100, d: 100, c: 100));    
}
```

### 二、匿名函数(闭包，lambda)

在Dart中可以创建一个没有函数名称的函数，这种函数称为**匿名函数，或者lambda函数或者闭包函数**。但是和其他函数一样，它也有形参列表，可以有可选参数。

```dart
(num x) => x;//没有函数名，有必需的位置参数x
(num x) {return x;}//等价于上面形式
(int x, [int step]) => x + step;//没有函数名，有可选的位置参数step
(int x, {int step1, int step2}) => x + step1 + step2;////没有函数名，有可选的命名参数step1、step2
```

### 闭包在dart中的应用

闭包函数在dart用的特别多，单从集合中操作符来说就有很多。

```dart
main() {
  List<int> numbers = [3, 1, 2, 7, 12, 2, 4];
  //reduce函数实现累加，reduce函数中接收的(prev, curr) => prev + curr就是一个闭包
  print(numbers.reduce((prev, curr) => prev + curr));
  //还可以不用闭包形式来写，但是这并不是一个好的方案,不建议下面这样使用。
  plus(prev, curr) => prev + curr;
  print(numbers.reduce(plus));
}
//reduce函数定义
 E reduce(E combine(E value, E element)) {//combine闭包函数
    Iterator<E> iterator = this.iterator;
    if (!iterator.moveNext()) {
      throw IterableElementError.noElement();
    }
    E value = iterator.current;
    while (iterator.moveNext()) {
      value = combine(value, iterator.current);//执行combine函数
    }
    return value;
  }
```

### 三、箭头函数

在Dart中还有一种函数的简写形式，那就是**箭头函数**。箭头函数**是只能包含一行表达式**的函数，会注意到它**没有花括号，而是带有箭头**的。箭头函数更有助于代码的可读性，类似于Kotlin或Java中的lambda表达式`->`的写法。

```dart
main() {
  List<int> numbers = [3, 1, 2, 7, 12, 2, 4];
  print(numbers.reduce((prev, curr) {//闭包简写形式
        return prev + curr;
  }));
  print(numbers.reduce((prev, curr) => prev + curr)); //等价于上述形式，箭头函数简写形式
}
```

### 四、局部函数

在Dart中还有一种可以直接定义在函数体内部的函数，可以把称为**局部函数或者内嵌函数**。我们知道函数声明可以出现顶层，比如常见的main函数等等。局部函数的好处就是从作用域角度来看，它可以访问外部函数变量，并且还能避免引入一个额外的外部函数，使得整个函数功能职责统一。

```dart
//定义外部函数fibonacci
int fibonacci(int n) {
    //定义局部函数lastTwo
    List<int> lastTwo(int n) {
        if(n < 1) {
           return <int>[0, 1];  
        } else {
           var p = lastTwo(n - 1);
           return <int>[p[1], p[0] + p[1]];
        }
    }
    return lastTwo(n)[1];
}
```

### 五、顶层函数和静态函数

在Dart中有一种特别的函数，我们知道在面向对象语言中比如Java，并不能直接定义一个函数的，而是需要定义一个类，然后在类中定义函数。但是在Dart中可以不用在类中定义函数，而是直接基于dart文件顶层定义函数，这种函数我们一般称为**顶层函数**。最常见就是main函数了。而静态函数就和Java中类似，依然使用**static关键字来声明**，然后必须是**定义在类的内部**的。

```dart
//顶层函数，不定义在类的内部
main() {
  print('hello dart');
}

class Number {
    static int getValue() => 100;//static修饰定义在类的内部。
}
```

### 六、main函数

每个应用程序都有一个顶级的main()函数，它作为应用程序的入口点。main()函数返回void，所以在dart可以直接省略void，并有一个可选的列表参数作为参数。

```dart
//你一般看到的main是这样的
main() {
  print('hello dart');
}
//实际上它和Java类似可以带个参数列表
main(List<String> args) {
  print('hello dart: ${args[0]}, ${args[1]}');//用dart command执行的时候: dart test.dart arg0 arg1 =>输出:hello dart: arg0, arg1    
}
```

### 七、Function函数对象

在Dart中一切都是对象，函数也不例外，函数可以作为一个参数传递。其中`Function`类是代表所有函数的公共顶层接口抽象类。`Function`类中并没有声明任何实例方法。但是它有一个非常重要的静态类函数`apply`. 该函数接收一个`Function`对象`function`，一个`List`的参数`positionalArguments`以及一个可选参数`Map<Symbol, dynamic>`类型的`namedArguments`。大家似乎明白了什么？知道为啥dart中函数支持位置参数和命名参数吗? 没错就是它们两个参数功劳。实际上，**apply()函数提供一种使用动态确定的参数列表来调用函数的机制，通过它我们就能处理在编译时参数列表不确定的情况**。

```dart
abstract class Function {
  external static apply(Function function, List positionalArguments,
      [Map<Symbol, dynamic> namedArguments]);//可以看到这是external声明，我们需要找到对应的function_patch.dart实现

  int get hashCode;

  bool operator ==(Object other);
}
```

在sdk源码中找到`sdk/lib/_internal/vm/lib/function_patch.dart`对应的`function_patch`的实现

```dart
@patch
class Function {
  // TODO(regis): Pass type arguments to generic functions. Wait for API spec.
  //可以看到内部私有的_apply函数，最终接收两个List原生类型的参数arguments,names分别代表着我们使用函数时
  //定义的所有参数List集合arguments(包括位置参数和命名参数)以及命名参数名List集合names，不过它是委托到native层的Function_apply C++函数实现的。
  static _apply(List arguments, List names) native "Function_apply";

  @patch
  static apply(Function function, List positionalArguments,
      [Map<Symbol, dynamic> namedArguments]) {
    //计算外部函数位置参数的个数  
    int numPositionalArguments = 1 + // 默认同时会传入function参数，所以默认+1
        (positionalArguments != null ? positionalArguments.length : 0);//位置参数的集合不为空就返回集合长度否则返回0
    //计算外部函数命名参数的个数    
    int numNamedArguments = namedArguments != null ? namedArguments.length : 0;;//命名参数的集合不为空就返回集合长度否则返回0
    //计算所有参数个数总和: 位置参数个数 + 命名参数个数
    int numArguments = numPositionalArguments + numNamedArguments;
    //创建一个定长为所有参数个数大小的List集合arguments
    List arguments = new List(numArguments);
    //集合第一个元素默认是传入的function对象
    arguments[0] = function;
    //然后从1的位置开始插入所有的位置参数到arguments参数列表中
    arguments.setRange(1, numPositionalArguments, positionalArguments);
    //然后再创建一个定长为命名参数长度的List集合
    List names = new List(numNamedArguments);
    int argumentIndex = numPositionalArguments;
    int nameIndex = 0;
    //遍历命名参数Map集合
    if (numNamedArguments > 0) {
      namedArguments.forEach((name, value) {
        arguments[argumentIndex++] = value;//把命名参数对象继续插入到arguments集合中
        names[nameIndex++] = internal.Symbol.getName(name);//并把对应的参数名标识存入names集合中
      });
    }
    return _apply(arguments, names);//最后调用_apply函数传入所有参数对象集合以及命名参数名称集合
  }
}
```

不妨再来瞅瞅C++层中的`Function_apply`的实现

```cpp
DEFINE_NATIVE_ENTRY(Function_apply, 0, 2) {
  const int kTypeArgsLen = 0;  // TODO(regis): Add support for generic function.
  const Array& fun_arguments =
      Array::CheckedHandle(zone, arguments->NativeArgAt(0));//获取函数的所有参数对象数组 fun_arguments
  const Array& fun_arg_names =
      Array::CheckedHandle(zone, arguments->NativeArgAt(1));//获取函数的命名参数参数名数组 fun_arg_names
  const Array& fun_args_desc = Array::Handle(
      zone, ArgumentsDescriptor::New(kTypeArgsLen, fun_arguments.Length(),
                                     fun_arg_names));//利用 fun_arg_names生成对应命名参数描述符集合
 //注意: 这里会调用DartEntry中的InvokeClosure函数，传入了所有参数对象数组 fun_arguments和fun_arg_names生成对应命名参数描述符集合
//最后返回result
  const Object& result = Object::Handle(
      zone, DartEntry::InvokeClosure(fun_arguments, fun_args_desc));
  if (result.IsError()) {
    Exceptions::PropagateError(Error::Cast(result));
  }
  return result.raw();
}
```

### 总结

到这里有关Dart中的函数就说完，有了这篇文章相信大家对dart函数应该有个全面的了解了。欢迎持续，下一篇我们将进入Dart中的面向对象...

### Dart系列文章，欢迎查看:

- **[Dart语法篇之集合操作符函数与源码分析(三)](https://link.zhihu.com/?target=https%3A//juejin.im/post/5dbc2265f265da4cf77c8898)**
- **[Dart语法篇之集合的使用与源码解析(二)](https://link.zhihu.com/?target=https%3A//juejin.im/post/5db9bad36fb9a02063699df6)**
- **[Dart语法篇之基础语法(一)](https://link.zhihu.com/?target=https%3A//juejin.im/post/5dac7b93e51d45248c7b5182)**