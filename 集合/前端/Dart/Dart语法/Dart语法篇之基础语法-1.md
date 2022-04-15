# [Dart语法篇之基础语法(一)](https://zhuanlan.zhihu.com/p/88728224)

简述:

又是一段新的开始，Dart这门语言相信很多人都是通过Flutter这个框架才了解的，因为Flutter相比Dart更被我们所熟知。很多人迟迟不愿尝试Flutter原因大多数是因为学习成本高，显然摆在面前的是需要去重新学习一门新的语言dart，然后再去学习一个开发框架Flutter,再加上很多莫名奇妙的坑,不说多的就从Github上Flutter项目issue数来看坑是着实不少，所以很多人也就望而却步了。当然这个问题是一个新的技术框架刚开始都可能存在的，但我们更需要看到Flutter框架跨平台技术思想先进性。

为什么要开始一系列Dart相关的文章?

很简单，就是为了更好地开发Flutter, 其实开发Flutter使用Dart的核心知识点并不需要太过于全面，有些东西根本用不到，所以该系列文章也将是有选择性选取Dart一些常用技术点讲解。另一方面，读过我文章的小伙伴就知道我是Kotlin的狂热忠实粉, 其实语言都是相通，你可以从Dart身上又能看到Kotlin的身影，所以我上手Dart非常快，可以对比着学习。所以后期Dart文章我会将Dart与Kotlin作为对比讲解，所以如果你学过Kotlin那么恭喜你，上手Dart将会非常快。

该系列Dart文章会讲哪些内容呢?

本系列文章主要会涉及以下内容: dart基本语法、变量常量和类型推导、集合、函数、面向对象的Mixins、泛型、生成器函数、Async和Await、Stream和Future、Isolate和EventLoop以及最后基本介绍下DartVM的工作原理。

### 一、Hello Dart

> 这是第一个Hello Dart程序，很多程序入口都是从main函数开始，所以dart也不例外，一起来看下百变的main函数

```dart
//main标准写法
void main() {
  print('Hello World!');//注意: Dart和Java一样表达式以分号结尾，写习惯Kotlin的小伙伴需要注意了, 这可能是你从Kotlin转Dart最大不适之一。
}

//dart中void类型，作为函数返回值类型可以省略
main() {
  print('Hello World!');  
}

//如果函数内部只有一个表达式，可以省略大括号，使用"=>"箭头函数; 
//而对于Kotlin则是如果只有一个表达式，可以省略大括号，使用"="连接，类似 fun main(args: Array<String>) = println('Hello World!')
void main() => print('Hello World!');

//最简写形式
main() => print('Hello World!');
```

### 二、数据类型

在dart中的一切皆是对象，包括数字、布尔值、函数等，它们和Java一样都继承于Object, 所以它们的默认值也就是null. 在dart主要有: 布尔类型bool、数字类型num(数字类型又分为int，double，并且两者父类都是num)、字符串类型String、集合类型(List, Set, Map)、Runes类和Symbols类型(后两个用的并不太多)

### 1、布尔类型(bool)

在dart中和C语言一样都是使用`bool`来声明一个布尔类型变量或常量，而在Kotlin则是使用`Boolean` 来声明，但是一致的是它对应的值只有两个true和false.

```dart
main() {
    bool isClosed = true;//注意，dart还是和Java类似的 [类型][变量名]方式声明，这个和Kotlin的 [变量名]:[类型]不一样.
    bool isOpened = false;
}
```

### 2、数字类型(num、int、double)

在dart中num、int、double都是类,然后int、double都继承num抽象类，这点和Kotlin很类似，在Kotlin中Number、Int、Double都是类，然后Int、Double都继承于Number. 注意，**但是在dart中没有float, short, long类型**

![](/Users/mikyou/Library/Application Support/marktext/images/2019-10-24-22-02-40-image.png)

```dart
main() {
    double pi = 3.141592653;
    int width = 200;
    int height = 300;
    print(width / height);//注意:这里和Kotlin、Java都不一样，两个int类型相除是double类型小数，而不是整除后的整数。
    print(width ~/ height);//注意: 这才是dart整除正确姿势
}
```

此外和Java、Kotlin一样，dart也拥有一些数字常用的函数:

```dart
main() {
    print(3.141592653.toStringAsFixed(3)); //3.142 保留有效数字
    print(6.6.floor());//6向下取整
    print((-6.6).ceil()); //-6 向上取整
    print(9.9.ceil()); //10 向上取整
    print(666.6.round()); //667 四舍五入
    print((-666.6).abs()); // 666.6 取绝对值
    print(666.6.toInt()); //666 转化成int,这中toInt、toDouble和Kotlin类似
    print(999.isEven); //false 是否是偶数
    print(999.isOdd); //true 是否是奇数
    print(666.6.toString()); //666.6 转化成字符串
}
```

### 3、字符串类型(String)

在Dart中支持单引号、双引号、三引号以及$字符串模板用法和Kotlin是一模一样的。

```dart
main() {
    String name = 'Hello Dart!';//单引号
    String title = "'Hello Dart!'";//双引号
    String description = """
          Hello Dart! Hello Dart!
          Hello Dart!
          Hello Dart! Hello Dart!
    """;//三引号
    num value = 2;
    String result = "The result is $value";//单值引用
    num width = 200;
    num height = 300;
    String square = "The square is ${width * height}";//表达式的值引用
}
```

和Kotlin一样，dart中也有很多字符串操作的方法，比如字符串拆分、子串等

```dart
main() {
  String url = "https://mrale.ph/dartvm/";

  print(url.split("://")[0]); //字符串分割split方法，类似Java和Kotlin

  print(url.substring(3, 9)); //字符串截取substring方法, 类似Java和Kotlin

  print(url.codeUnitAt(0)); //取当前索引位置字符的UTF-16码

  print(url.startsWith("https")); //当前字符串是否以指定字符开头, 类似Java和Kotlin

  print(url.endsWith("/")); //当前字符串是否以指定字符结尾, 类似Java和Kotlin

  print(url.toUpperCase()); //大写, 类似Java和Kotlin

  print(url.toLowerCase()); //小写, 类似Java和Kotlin

  print(url.indexOf("ph")); //获取指定字符的索引位置, 类似Java和Kotlin

  print(url.contains("http")); //字符串是否包含指定字符, 类似Java和Kotlin

  print(url.trim()); //去除字符串的首尾空格, 类似Java和Kotlin

  print(url.length); //获取字符串长度

  print(url.replaceFirst("t", "A")); //替换第一次出现t字符位置的字符

  print(url.replaceAll("m", "M")); //全部替换, 类似Java和Kotlin
}
```

### 4、类型检查(is和is!)和强制类型转换(as)

和Kotlin一样，dart也是通过 **is** 关键字来对类型进行检查以及使用 **as** 关键字对类型进行强制转换，如果判断不是某个类型dart中使用 **is!** , 而在Kotlin中正好相反则用 **!is** 表示。

```dart
main() {
    int number = 100;
    double distance = 200.5;
    num age = 18;
    print(number is num);//true
    print(distance is! int);//true
    print(age as int);//18
}
```

### 5、Runes和Symbols类型

在Dart中的Runes和Symbols类型使用并不多，这里做个简单的介绍, Runes类型是UTF-32字节单元定义的Unicode字符串，Unicode可以使用数字表示字母、数字和符号，然而在dart中String是一系列的UTF-16的字节单元，所以想要表示32位的Unicode的值，就需要用到Runes类型。我们一般使用`\uxxxx`这种形式来表示一个Unicode码，`xxxx` 表示4个十六进制值。当十六进制数据多余或者少于4位时，将十六进制数放入到花括号中，例如，微笑表情（ ）是`\u{1f600}`。而Symbols类型则用得很少，一般用于Dart中的反射，但是注意在Flutter中禁止使用反射。

```dart
main() {
  var clapping = '\u{1f44f}';
  print(clapping);
  print(clapping.codeUnits);//返回十六位的字符单元数组
  print(clapping.runes.toList());

  Runes input = new Runes(
      '\u2665  \u{1f605}  \u{1f60e}  \u{1f47b}  \u{1f596}  \u{1f44d}');
  print(new String.fromCharCodes(input));
}
```

### 6、Object类型

在Dart中所有东西都是对象，都继承于Object, 所以可以使用Object可以定义任何的变量，而且赋值后，类型也可以更改。

```dart
main() {
    Object color = 'black';
    color = 0xff000000;//运行正常，0xff000000类型是int, int也继承于Object   
}
```

### 7、dynamic类型

在Dart中还有一个和Object类型非常类似的类型那就是dynamic类型，下面讲到的var声明的变量未赋值的时候就是dynamic类型， 它可以像Object一样可以改变类型。dynamic类型一般用于无法确定具体类型, 注意: **建议不要滥用dynamic，一般尽量使用Object**, 如果你对Flutter和Native原生通信PlatformChannel代码熟悉的话，你会发现里面大量使用了dynamic, 因为可能native数据类型无法对应dart中的数据类型,此时dart接收一般就会使用dynamic.

Object和dynamic区别在于: Object会在**编译阶段**检查类型，而dynamic不会在**编译阶段**检查类型。

```dart
main() {
    dynamic color = 'black';
    color = 0xff000000;//运行正常，0xff000000类型是int, int也继承于Object
}
```

### 三、变量和常量

### 1、var关键字

在dart中可以使用var来替代具体类型的声明，会**自动推导变量的类型**，这是因为var并不是直接存储值，而是存储值的对象引用，所以var可以声明任何变量。这一点和Kotlin不一样，在Kotlin中声明可变的变量都必须需要使用var关键字，而Kotlin的类型推导是默认行为和var并没有直接关系。注意: 在Flutter开发一般会经常使用var声明变量，以便于可以自动推导变量的类型。

```dart
main() {
  int colorValue = 0xff000000;
  var colorKey = 'black'; //var声明变量 自动根据赋值的类型，推导为String类型 
  // 使用var声明集合变量 
  var colorList = ['red', 'yellow', 'blue', 'green'];
  var colorSet = {'red', 'yellow', 'blue', 'green'};
  var colorMap = {'white': 0xffffffff, 'black': 0xff000000};
}
```

但是在使用var声明变量的时候，需要注意的是: **如果var声明的变量开始不初始化，不仅值可以改变它的类型也是可以被修改的，但是一旦开始初始化赋值后，它的类型就确定了，后续不能被改变。**

```dart
main() {
  var color; // 仅有声明未赋值的时候，这里的color的类型是dynamic,所以它的类型是可以变的 
  color = 'red';
  print(color is String); //true 
  color = 0xffff0000;
  print(color is int); //true 

  var colorValue = 0xffff0000; //声明时并赋值，这里colorValue类型已经推导出为int,并且确定了类型 
  colorValue = 'red'; //错误，这里会抛出编译异常，String类型的值不能赋值给int类型 
  print(colorValue is int); //true
}
```

### 2、常量(final和const)

在dart中声明常量可以使用**const**或**final** 两个关键字，注意: 这两者的区别在于如果常量是编译期就能初始化的就用const(有点类似Kotlin中的const val) 如果常量是运行时期初始化的就用final(有点类似Kotlin中的val)

```dart
main() {    
  const PI = 3.141592653;//const定义常量    
  final nowTime = DateTime.now();//final定义常量
}
```

### 四、集合(List、Set、Map)

### 1、集合List

在dart中的List和Kotlin还是很大的区别，换句话说Dart整个集合类型系统的划分都和Kotlin都不一样，比如Dart中集合就没有严格区分成可变集合(Kotlin中MutableList)和不变集合(Kotlin中的List)，在使用方式上你会感觉它更像数组，但是它是可以随意对元素增删改成的。

- List初始化方式

```text
main() {       
   List<String> colorList = ['red', 'yellow', 'blue', 'green'];//直接使用[]形式初始化       
   var colorList = <String> ['red', 'yellow', 'blue', 'green'];   
}
```

- List常用的函数

```text
main() { 
   List<String> colorList = ['red', 'yellow', 'blue', 'green'];       
   colorList.add('white');//和Kotlin类似通过add添加一个新的元素       
   print(colorList[2]);//可以类似Kotlin一样，直接使用数组下标形式访问元素       
   print(colorList.length);//获取集合的长度，这个Kotlin不一样，Kotlin中使用的是size       
   colorList.insert(1, 'black');//在集合指定index位置插入指定的元素       
   colorList.removeAt(2);//移除集合指定的index=2的元素，第3个元素       
   colorList.clear();//清除所有元素       
   print(colorList.sublist(1,3));//截取子集合       
   print(colorList.getRange(1, 3));//获取集合中某个范围元素       
   print(colorList.join('<--->'));//类似Kotlin中的joinToString方法，输出: red<--->yellow<--->blue<--->green       
   print(colorList.isEmpty);       
   print(colorList.contains('green'));       
} 
```

- List的遍历方式

```text
main() {
   List<String> colorList = ['red', 'yellow', 'blue', 'green'];//for-i遍历       
   for(var i = 0; i < colorList.length; i++) {//可以使用var或int           
       print(colorList[i]);               
   }       
  //forEach遍历       
  colorList.forEach((color) => print(color));//forEach的参数为Function. =>使用了箭头函数       
  //for-in遍历       
  for(var color in colorList) {
      print(color);       
  }       
  //while+iterator迭代器遍历，类似Java中的iteator       
  while(colorList.iterator.moveNext()) {           
      print(colorList.iterator.current);       
  }   
} 
```

### 2、集合Set

集合Set和列表List的区别在于 **集合中的元素是不能重复** 的。所以添加重复的元素时会返回false,表示添加不成功.

- Set初始化方式

```text
main() {       
   Set<String> colorSet= {'red', 'yellow', 'blue', 'green'};//直接使用{}形式初始化       
   var colorList = <String> {'red', 'yellow', 'blue', 'green'};   
}
```

- 集合中的交、并、补集，在Kotlin并没有直接给到计算集合交、并、补的API

```dart
main() {       
   var colorSet1 = {'red', 'yellow', 'blue', 'green'};       
   var colorSet2 = {'black', 'yellow', 'blue', 'green', 'white'};       
   print(colorSet1.intersection(colorSet2));//交集-->输出: {'yellow', 'blue', 'green'}       
   print(colorSet1.union(colorSet2));//并集--->输出: {'black', 'red', 'yellow', 'blue', 'green', 'white'}       
   print(colorSet1.difference(colorSet2));//补集--->输出: {'red'}   
}
```

- Set的遍历方式(和List一样)

```dart
main() {       
   Set<String> colorSet = {'red', 'yellow', 'blue', 'green'};       
   //for-i遍历       
   for (var i = 0; i < colorSet.length; i++) {         
       //可以使用var或int         
       print(colorSet[i]);       
   }       
   //forEach遍历       
   colorSet.forEach((color) => print(color)); //forEach的参数为Function. =>使用了箭头函数       
   //for-in遍历       
   for (var color in colorSet) {         
       print(color);       
   }       
   //while+iterator迭代器遍历，类似Java中的iteator       
   while (colorSet.iterator.moveNext()) {         
       print(colorSet.iterator.current);       
   }     
}
```

### 3、集合Map

集合Map和Kotlin类似，key-value形式存储，并且 **Map对象的中key是不能重复的**

- Map初始化方式

```dart
main() {       
   Map<String, int> colorMap = {'white': 0xffffffff, 'black':0xff000000};//使用{key:value}形式初始化    
   var colorMap = <String, int>{'white': 0xffffffff, 'black':0xff000000};   
}
```

- Map中常用的函数

```dart
main() {       
   Map<String, int> colorMap = {'white': 0xffffffff, 'black':0xff000000};       
   print(colorMap.containsKey('green'));//false       
   print(colorMap.containsValue(0xff000000));//true       
   print(colorMap.keys.toList());//['white','black']       
   print(colorMap.values.toList());//[0xffffffff, 0xff000000]       
   colorMap['white'] = 0xfffff000;//修改指定key的元素       
   colorMap.remove('black');//移除指定key的元素   
}
```

- Map的遍历方式

```dart
main() {       
   Map<String, int> colorMap = {'white': 0xffffffff, 'black':0xff000000};       
   //for-each key-value       
   colorMap.forEach((key, value) => print('color is $key, color value is $value'));   
}
```

- Map.fromIterables将List集合转化成Map

```dart
main() {       
   List<String> colorKeys = ['white', 'black'];       
   List<int> colorValues = [0xffffffff, 0xff000000];       
   Map<String, int> colorMap = Map.fromIterables(colorKeys, colorValues);   
} 
```

### 4、集合常用的操作符

dart对于集合操作的也非常符合现代语言的特点，含有丰富的集合操作符API，可以让你处理结构化的数据更加简单。

```dart
main() {
  List<String> colorList = ['red', 'yellow', 'blue', 'green'];
  //forEach箭头函数遍历
  colorList.forEach((color) => {print(color)});
  colorList.forEach((color) => print(color)); //箭头函数遍历，如果箭头函数内部只有一个表达式可以省略大括号

  //map函数的使用
  print(colorList.map((color) => '$color_font').join(","));

  //every函数的使用，判断里面的元素是否都满足条件，返回值为true/false
  print(colorList.every((color) => color == 'red'));

  //sort函数的使用
  List<int> numbers = [0, 3, 1, 2, 7, 12, 2, 4];
  numbers.sort((num1, num2) => num1 - num2); //升序排序
  numbers.sort((num1, num2) => num2 - num1); //降序排序
  print(numbers);

  //where函数使用，相当于Kotlin中的filter操作符，返回符合条件元素的集合
  print(numbers.where((num) => num > 6));

  //firstWhere函数的使用，相当于Kotlin中的find操作符，返回符合条件的第一个元素，如果没找到返回null
  print(numbers.firstWhere((num) => num == 5, orElse: () => -1)); //注意: 如果没有找到，执行orElse代码块，可返回一个指定的默认值

  //singleWhere函数的使用，返回符合条件的第一个元素，如果没找到返回null，但是前提是集合中只有一个符合条件的元素, 否则就会抛出异常
  print(numbers.singleWhere((num) => num == 4, orElse: () => -1)); //注意: 如果没有找到，执行orElse代码块，可返回一个指定的默认值

  //take(n)、skip(n)函数的使用，take(n)表示取当前集合前n个元素, skip(n)表示跳过前n个元素，然后取剩余所有的元素
  print(numbers.take(5).skip(2));

  //List.from函数的使用，从给定集合中创建一个新的集合,相当于clone一个集合
  print(List.from(numbers));

  //expand函数的使用, 将集合一个元素扩展成多个元素或者将多个元素组成二维数组展开成平铺一个一位数组
  var pair = [
    [1, 2],
    [3, 4]
  ];
  print('flatten list: ${pair.expand((pair) => pair)}');

  var inputs = [1, 2, 3];
  print('duplicated list: ${inputs.expand((number) =>[
    number,
    number,
    number
  ])}');
}
```

### 五、流程控制

### 1、for循环

```dart
main() {
    List<String> colorList = ['red', 'yellow', 'blue', 'green'];
    for (var i = 0; i < colorList.length; i++) {//可以用var或int
        print(colorList[i]);
    }
}
```

### 2、while循环

```dart
main() {
    List<String> colorList = ['red', 'yellow', 'blue', 'green'];
    var index = 0;
    while (index < colorList.length) {
        print(colorList[index++]);
    }
}
```

### 3、do-while循环

```dart
main() {
    List<String> colorList = ['red', 'yellow', 'blue', 'green'];
    var index = 0;
    do {
        print(colorList[index++]);
    } while (index < colorList.length);
}
```

### 4、break和continue

```dart
main() {
    List<String> colorList = ['red', 'yellow', 'blue', 'green'];
    for (var i = 0; i < colorList.length; i++) {//可以用var或int
        if(colorList[i] == 'yellow') {
            continue;
        }
        if(colorList[i] == 'blue') {
            break;
        }
        print(colorList[i]);
    }
}
```

### 5、if-else

```dart
void main() {
  var numbers = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11];
  for (var i = 0; i < numbers.length; i++) {
    if (numbers[i].isEven) {
      print('偶数: ${numbers[i]}');
    } else if (numbers[i].isOdd) {
      print('奇数: ${numbers[i]}');
    } else {
      print('非法数字');
    }
  }
}
```

### 6、三目运算符(? : )

```dart
void main() {
  var numbers = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11];
  for (var i = 0; i < numbers.length; i++) {
      num targetNumber = numbers[i].isEven ? numbers[i] * 2 : numbers[i] + 4;
      print(targetNumber);
  }
}
```

### 7、switch-case语句

```dart
Color getColor(String colorName) {
  Color currentColor = Colors.blue;
  switch (colorName) {
    case "read":
      currentColor = Colors.red;
      break;
    case "blue":
      currentColor = Colors.blue;
      break;
    case "yellow":
      currentColor = Colors.yellow;
      break;
  }
  return currentColor;
}
```

### 8、Assert(断言)

在dart中如果条件表达式结果不满足条件，则可以使用 `assert` 语句中断代码的执行。特别是在Flutter源码中随处可见都是assert断言的使用。注意: **断言只在检查模式下运行有效，如果在生产模式运行，则断言不会执行。**

```dart
assert(text != null);//text为null,就会中断后续代码执行
assert(urlString.startsWith('https'));
```

### 六、运算符

### 1、算术运算符

| 名称 | 运算符 | 例子 |

|:---:|:---:|:-------------------------:|

| 加 | + | var result = 1 + 1; |

| 减 | - | var result = 5 - 1; |

| 乘 | * | var result = 3 * 5; |

| 除 | / | var result = 3 / 5; [//0.6](https://link.zhihu.com/?target=https%3A//0.0.0.6/) |

| 整除 | ~/ | var result = 3 ~/ 5; //0 |

| 取余 | % | var result = 5 % 3; //2 |



### 2、条件运算符

| 名称 | 运算符 | 例子 |

|:----:|:---:|:------:|

| 大于 | > | 2 > 1 |

| 小于 | < | 1 < 2 |

| 等于 | == | 1 == 1 |

| 不等于 | != | 3 != 4 |

| 大于等于 | >= | 5 >= 4 |

| 小于等于 | <= | 4 <= 5 |

### 3、逻辑运算符

| 名称 | 运算符 | 例子 |

|:---:|:----:|:----------------:|

| 或 | \|\| | 2 > 1 \|\| 3 < 1 |

| 与 | && | 2 > 1 && 3 < 1 |

| 非 | ！ | !(2 > 1) |

### 4、位运算符

| 名称 | 运算符 |

|:---:|:---:|

| 位与 | & |

| 位或 | \| |

| 位非 | ~ |

| 异或 | ^ |

| 左移 | << |

| 右移 | >> |

### 5、三目运算符

**condition ? expr1 : expr2**

```dart
var isOpened = (value == 1) ? true : false;
```

### 6、空安全运算符

| 操作符 | 解释 |

|:-----------------------:|:---------------------------------------:|

| result = expr1 ?? expr2 | 若expr1为null, 返回expr2的值，否则返回expr1的值 |

| expr1 ??= expr2 | 若expr1为null, 则把expr2的值赋值给expr1 |

| result = expr1?.value | 若expr1为null, 就返回null,否则就返回expr1.value的值 |

- 1、**result = expr1 ?? expr2**

如果发现expr1为null,就返回expr2的值，否则就返回expr1的值, 这个类似于Kotlin中的 **result = expr1 ?: expr2**

```dart
  main() {
      var choice = question.choice ?? 'A';
      //等价于
      var choice2;
      if(question.choice == null) {
          choice2 = 'A';
      } else {
          choice2 = question.choice;
      }
  }
```



- 2、**expr1 ??= expr2** 等价于 **expr1 = expr1 ?? expr2** (转化成第一种)

```dart
  main() {
      var choice ??= 'A';
      //等价于
      if(choice == null) {
          choice = 'A';
      }
  }
```



- 3、**result = expr1?.value**

如果expr1不为null就返回expr1.value，否则就会返回null, 类似Kotlin中的 ?. 如果expr1不为null,就执行后者

```dart
var choice = question?.choice;   //等价于  
if(question == null){
   return null;   
} else {       
   return question.choice;   
}
question?.commit();   //等价于   
if(question == null){       
   return;//不执行commit()   
} else {       
  question.commit();//执行commit方法  
}   
```



### 7、级联操作符(..)

级联操作符是 `..`, 可以让你对一个对象中字段进行链式调用操作，类似Kotlin中的apply或run标准库函数的使用。

```dart
question
    ..id = '10001'
    ..stem = '第一题: xxxxxx'
    ..choices = <String> ['A','B','C','D']
    ..hint = '听音频做题';
```

Kotlin中的run函数实现对比

```kotlin
question.run {
    id = '10001'
    stem = '第一题: xxxxxx'
    choices = <String> ['A','B','C','D']
    hint = '听音频做题'    
}
```

### 8、运算符重载

在dart支持运算符自定义重载,使用**operator**关键字定义重载函数

```dart
class Vip {
  final int level;
  final int score;

  const Vip(this.level, this.score);

  bool operator >(Vip other) =>
      level > other.level || (level == other.level && score > other.score);

  bool operator <(Vip other) =>
      level < other.level || (level == other.level && score < other.score);

  bool operator ==(Vip other) =>
      level == other.level &&
      score == other.level; //注意: 这段代码可能在高版本的Dart中会报错，在低版本是OK的
  //上述代码，在高版本Dart中，Object中已经重载了==,所以需要加上covariant关键字重写这个重载函数。
  @override
  bool operator ==(covariant Vip other) =>
      (level == other.level && score == other.score);

  @override
  int get hashCode => super.hashCode; //伴随着你还需要重写hashCode，至于什么原因大家应该都知道
}


main() {
    var userVip1 = Vip(4, 3500);
    var userVip2 = Vip(4, 1200);
    if(userVip1 > userVip2) {
        print('userVip1 is super vip');
    } else if(userVip1 < userVip2) {
        print('userVip2 is super vip');
    }
}
```

### 七、异常

dart中的异常捕获方法和Java,Kotlin类似，使用的也是**try-catch-finally**; 对特定异常的捕获使用**on**关键字. dart中的常见异常有: **NoSuchMethodError**(当在一个对象上调用一个该对象没有 实现的函数会抛出该错误)、**ArgumentError** (调用函数的参数不合法会抛出这个错误)

```dart
main() {
  int num = 18;
  int result = 0;
  try {
    result = num ~/ 0;
  } catch (e) {//捕获到IntegerDivisionByZeroException
    print(e.toString());
  } finally {
    print('$result');
  }
}

//使用on关键字捕获特定的异常
main() {
  int num = 18;
  int result = 0;
  try {
    result = num ~/ 0;
  } on IntegerDivisionByZeroException catch (e) {//捕获特定异常
    print(e.toString());
  } finally {
    print('$result');
  }
}
```

### 八、函数

在dart中函数的地位一点都不亚于对象，支持闭包和高阶函数，而且dart中的函数也会比Java要灵活的多，而且Kotlin中的一些函数特性，它也支持甚至比Kotlin支持的更全面。比如支持默认值参数、可选参数、命名参数等.

### 1、函数的基本用法

```dart
main() {
    print('sum is ${sum(2, 5)}');
}

num sum(num a, num b) {
    return a + b;
}
```

### 2、函数参数列表传参规则

```dart
//num a, num b, num c, num d 最普通的传参: 调用时，参数个数和参数顺序必须固定
add1(num a, num b, num c, num d) {
  print(a + b + c + d);
}

//[num a, num b, num c, num d]传参: 调用时，参数个数不固定，但是参数顺序需要一一对应, 不支持命名参数
add2([num a, num b, num c, num d]) {
  print(a + b + c + d);
}

//{num a, num b, num c, num d}传参: 调用时，参数个数不固定，参数顺序也可以不固定，支持命名参数,也叫可选参数，是dart中的一大特性，这就是为啥Flutter代码那么多可选属性，大量使用可选参数
add3({num a, num b, num c, num d}) {
  print(a + b + c + d);
}

//num a, num b, {num c, num d}传参: 调用时，a,b参数个数固定顺序固定，c,d参数个数和顺序也可以不固定
add4(num a, num b, {num c, num d}) {
  print(a + b + c + d);
}

main() {
  add1(100, 100, 100, 100); //最普通的传参: 调用时，参数个数和参数顺序必须固定
  add2(100, 100); //调用时，参数个数不固定，但是参数顺序需要一一对应, 不支持命名参数(也就意味着顺序不变)
  add3(
      b: 200,
      a: 200,
      c: 100,
      d: 100); //调用时，参数个数不固定，参数顺序也可以不固定，支持命名参数(也就意味着顺序可变)
  add4(100, 100, d: 100, c: 100); //调用时，a,b参数个数固定顺序笃定，c,d参数个数和顺序也可以不固定
}
```

### 3、函数默认参数和可选参数(以及与Kotlin对比)

dart中函数的默认值参数和可选参数和Kotlin中默认值参数和命名参数一致，只是写法上不同而已

```dart
add3({num a, num b, num c, num d = 100}) {//d就是默认值参数，给的默认值是100
   print(a + b + c + d);
}

main() {
    add3(b: 200, a: 100, c: 800);
}
```

与Kotlin对比

```kotlin
fun add3(a: Int, b: Int, c: Int, d: Int = 100) {
    println(a + b + c + d)
}

fun main(args: Array<String>) {
    add3(b = 200, a = 100, c = 800)
}
```

### 4、函数类型与高阶函数

在dart函数也是一种类型Function,可以作为函数参数传递，也可以作为返回值。类似Kotlin中的FunctionN系列函数

```dart
main() {
  Function square = (a) {
    return a * a;
  };

  Function square2 = (a) {
    return a * a * a;
  };

  add(3, 4, square, square2)
}

num add(num a, num b, [Function op, Function op2]) {
  //函数作为参数传递
  return op(a) + op2(b);
}
```

### 5、函数的简化以及箭头函数

在dart中的如果在函数体内只有一个表达式，那么就可以使用箭头函数来简化代码，这点也和Kotlin类似，只不过在Kotlin中人家叫lambda表达式，只是写法上不一样而已。

```dart
add4(num a, num b, {num c, num d}) {
  print(a + b + c + d);
}

add5(num a, num b, {num c, num d})  =>  print(a + b + c + d);
```

### 九、面向对象

在dart中一切皆是对象，所以面向对象在Dart中依然举足轻重，下面就先通过一个简单的例子认识下dart的面向对象，后续会继续深入。

### 1、类的基本定义和使用

```dart
abstract class Person {
    String name;
    int age;
    double height;
    Person(this.name, this.age, this.height);//注意，这里写法可能大家没见过， 这点和Java是不一样，这里实际上是一个dart的语法糖。但是这里不如Kotlin，Kotlin是直接把this.name传值的过程都省了。
    //与上述的等价代码,当然这也是Java中必须要写的代码
    Person(String name, int age, double height) {
        this.name = name;
        this.age = age;
        this.height = height;
    }   
    //然而Kotlin很彻底只需要声明属性就行,下面是Kotlin实现代码
    abstract class Person(val name: String, val age: Int, val height: Double)     
}

class Student extends Person {//和Java一样同时使用extends关键字表示继承
    Student(String name, int age, double height, double grade): super(name, age, height);//在 Dart里：类名(变量，变量,...) 是构造函数的写法, :super()表示该构造调用父类，这里构造时传入三个参数
}
```

### 2、类中属性的getter和setter访问器(类似Kotlin)

```dart
abstract class Person {
  String _name; ////相当于kotlin中的var 修饰的变量有setter、getter访问器，在dart中没有访问权限, 默认_下划线开头变量表示私有权限，外部文件无法访问
  final int _age;//相当于kotlin中的val 修饰的变量只有getter访问器
  Person(this._name, this._age); //这是上述简写形式

  //使用set关键字 计算属性 自定义setter访问器
  set name(String name) => _name = name;
  //使用get关键字 计算属性 自定义getter访问器
  bool get isStudent => _age > 18;
}
```

### 总结

这是dart的第一篇文章，主要就是从整体上介绍了下dart的语法，当然里面还有一些东西需要深入，后续会继续深入探讨。整体看下有没有觉得Kotlin和dart语法很像，其实里面有很多特性都是现代编程语言的特性，包括你在其他语言中同样能看到比如swift等。就到这里，后面继续聊dart...