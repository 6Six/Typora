# Dart语法篇之面向对象基础(五)

简述:

从这篇文章开始，我们继续Dart语法篇的第五讲, dart中的面向对象基础。我们知道在Dart中一切都是对象，所以面向对象在Dart开发中是非常重要的。此外它还和其他有点不一样的地方，比如多继承mixin、构造器不能被重载、setter和getter的访问器函数等。

### 一、属性访问器(accessor)函数setter和getter

在Dart类的属性中有一种为了方便访问它的值特殊函数，那就是setter,getter属性访问器函数。**实际上，在dart中每个实例属性始终有与之对应的setter,getter函数(若是final修饰只读属性只有getter函数, 而可变属性则有setter，getter两种函数)。而在给实例属性赋值或获取值时，实际上内部都是对setter和getter函数的调用**。

### 1、属性访问器函数setter

setter函数名前面添加**前缀`set`**, 并只接收一个参数。setter调用语法于传统的变量赋值是一样的。如果一个实例属性是可变的，那么一个setter属性访问器函数就会为它自动定义，所有实例属性的赋值实际上都是对setter函数的调用。这一点和Kotlin中的setter，getter非常相似。

```dart
class Rectangle {
  num left, top, width, height;

  Rectangle(this.left, this.top, this.width, this.height);
  set right(num value) => left = value - width;//使用set作为前缀，只接收一个参数value
  set bottom(num value) => top = value - height;//使用set作为前缀，只接收一个参数value
}

main() {
  var rect = Rectangle(3, 4, 20, 15);
  rect.right = 15;//调用setter函数时，可以直接使用类似属性赋值方式调用right函数。
  rect.bottom = 12;//调用setter函数时，可以直接使用类似属性赋值方式调用bottom函数。
}
```

对比Kotlin中的实现

```kotlin
class Rectangle(var left: Int, var top: Int, var width: Int, var height: Int) {
    var right: Int = 0//在kotlin中表示可变使用var,只读使用val
        set(value) {//kotlin中定义setter
            field = value
            left = value - width
        }
    var bottom: Int = 0
        set(value) {//kotlin中定义setter
            field = value
            top = value - height
        }
}

fun main(args: Array<String>) {
    val rect = Rectangle(3, 4, 20, 15);
    rect.right = 15//调用setter函数时，可以直接使用类似属性赋值方式调用right函数。
    rect.bottom = 12//调用setter函数时，可以直接使用类似属性赋值方式调用bottom函数。
}
```

### 2、属性访问器函数getter

Dart中所有实例属性的访问都是通过调用getter函数来实现的。每个实例数额行始终都有一个与之关联的getter,由Dart编译器提供的。

```dart
class Rectangle {
  num left, top, width, height;

  Rectangle(this.left, this.top, this.width, this.height);
  get square => width * height; ;//使用get作为前缀，getter来计算面积.
  set right(num value) => left = value - width;//使用set作为前缀，只接收一个参数value
  set bottom(num value) => top = value - height;//使用set作为前缀，只接收一个参数value
}

main() {
  var rect = Rectangle(3, 4, 20, 15);
  rect.right = 15;//调用setter函数时，可以直接使用类似属性赋值方式调用right函数。
  rect.bottom = 12;//调用setter函数时，可以直接使用类似属性赋值方式调用bottom函数。
  print('the rect square is ${rect.square}');//调用getter函数时，可以直接使用类似读取属性值方式调用square函数。
}
```

对比kotlin实现

```kotlin
class Rectangle(var left: Int, var top: Int, var width: Int, var height: Int) {
    var right: Int = 0
        set(value) {
            field = value
            left = value - width
        }
    var bottom: Int = 0
        set(value) {
            field = value
            top = value - height
        }

    val square: Int//因为只涉及到了只读，所以使用val
        get() = width * height//kotlin中定义getter
}

fun main(args: Array<String>) {
    val rect = Rectangle(3, 4, 20, 15);
    rect.right = 15
    rect.bottom = 12
    println(rect.square)//调用getter函数时，可以直接使用类似读取属性值方式调用square函数。
}
```

### 3、属性访问器函数使用场景

其实，上面`setter`，`getter`函数实现的目的，普通函数也能做到的。但是如果用`setter`,`getter`函数形式更符合编码规范。既然普通函数也能做到，那具体什么时候使用`setter`,`getter`函数，什么时候使用普通函数呢。这不得不把这个问题和另一问题转化一下成为: 哪种场景该定义属性还是定义函数的问题(关于这个问题，记得很久之前在讨论Kotlin的语法详细介绍过)。我们都知道**函数一般描述动作行为，而属性则是描述状态数据结构(状态可能经过多个属性值计算得到)。** 如果类中需要向外暴露类中某个状态那么更适合使用`setter`,`getter`函数；如果是触发类中的某个行为操作，那么普通函数更适合一点。

比如下面这个例子，`draw`绘制矩形动作更适合使用普通函数来实现，`square`获取矩形的面积更适合使用getter函数来实现,可以仔细体会下。

```dart
class Rectangle {
  num left, top, width, height;

  Rectangle(this.left, this.top, this.width, this.height);

  set right(num value) => left = value - width; //使用set作为前缀，只接收一个参数value
  set bottom(num value) => top = value - height; //使用set作为前缀，只接收一个参数value
  get square => width * height; //getter函数计算面积，描述Rectangle状态特性
  bool draw() {
    print('draw rect'); //draw绘制函数，触发是动作行为
    return true;
  }
}

main() {
  var rect = Rectangle(3, 4, 20, 15);
  rect.right = 15; //调用setter函数时，可以直接使用类似属性赋值方式调用right函数。
  rect.bottom = 12; //调用setter函数时，可以直接使用类似属性赋值方式调用bottom函数。
  print('the rect square is ${rect.square}');
  rect.draw();
}
```

### 二、面向对象中的变量

### 1、实例变量

实例变量实际上就是类的成员变量或者称为成员属性，当声明一个实例变量时，它会确保每一个对象实例都有自己唯一属性的拷贝。如果要表示实例私有属性的话就直接在属性名前面加下划线 **`_`**,例如`_width`和`_height`

```dart
class Rectangle {
  num left, top, _width, _height;//声明了left,top,_width,_height四个成员属性，未初始化时，它们的默认值都是null
}
```

上述例子中的`left, top, width, height`都是会自动引入一个`getter`和`setter`.事实上，在dart中属性都不是直接访问的，所有对字段属性的引用都是对属性访问器函数的调用, 只有访问器函数才能直接访问它的状态。

### 2、类变量(static变量)与顶层变量

**类变量**实际上就是`static`修饰的变量，属于类的作用域范畴；**顶层变量**就是**定义的变量不在某个具体类体内**，而是处于整个代码文件中，相当于文件顶层，和顶层函数差不多意思。`static`变量更多人愿意把它称为**静态变量**，但是在**Dart中静态变量不仅仅包括static变量还包括顶层变量**。

其实对于**类变量和顶层变量的访问都还是通过调用它的访问器函数来实现的**，但是类变量和顶层变量有点特殊，**它们是延迟初始化的**，在`getter`函数第一次被调用时类变量或顶层变量才执行初始化，也即是第一次引用类变量或顶层变量的时候。如果类变量或顶层变量没有被初始化默认值还是`null`.

```dart
class Animal {}
class Dog extends Animal {}
class Cat extends Animal {
    Cat() {
        print("I'm a Cat!");
    }
}
//注意,这里变量不定义在任何具体类体内，所以这个animal是一个顶层变量。
//虽然看似创建了Cat对象，但是由于顶层变量延迟初始化的原因，这里根本就没有创建Cat对象
Animal animal = Cat();
main() {
    animal = Dog();//然后又将animal引用指向了一个新的Dog对象，
}
```

顶层变量是具有延迟初始化过程，所以`Cat`对象并没有创建，因为整个代码执行中并没有去访问`animal`，所以无法触发第一次`getter`函数，也就导致`Cat`对象没有创建，直接表现是根本就不会输出`I'm a Cat!`这句话。这就是为什么顶层变量是延迟初始化的原因，`static`变量同理。

### 3、final 变量

在Dart中使用 **`final`** 关键字修饰变量，表示该变量初始化后不能再被修改。**`final`** 变量只有 **`getter`** 访问器函数，没有 **`setter`** 函数。类似于Kotlin中的`val`修饰的变量。声明成`final`的变量必须在实例方法运行前进行初始化，所以初始化`final`变量有很多中方法。注意: 建议尽量使用`final`来声明变量

```dart
class Person {
    final String gender = '男';//直接在声明的时候初始化，这种方式比较局限，针对基本数据类型还可以，但如果是一个对象类型就显示不合适了。
    final String name;
    final int age;
    Person(this.name, this.age);//利用构造函数为final变量初始化。
    //上述代码等价于下面实现
    Person(String name, int age) {
        this.name = name;
        this.age = age;
    }
}
```

**`final`与`const`的区别**，就好比Kotlin中的`val`与`const val`之间的区别，**`const`是编译期就进行了初始化**，而 **`final`则是运行期进行初始化**。

### 4、常量对象

在dart有些**对象是在编译期就可以计算的常量**，所以在dart中支持常量对象的定义，**常量对象**的创建需要使用 **`const`** 关键字。**常量对象的创建也是调用类的构造函数，但是注意必须是常量构造函数**，该构造函数是用 **`const`** 关键字修饰的。常量构造函数**必须是数字、布尔值或字符串，此外常量构造函数不能有函数体**，但是它可以有初始化列表。

```dart
class Point {
    final double x, y;
    const Point(this.x, this.y);//常量构造函数，使用const关键字且没有函数体
}

main() {
    const defaultPoint = const Point(0, 0);//创建常量对象
}
```

### 二、构造函数

### 1、主构造函数

**主构造函数**是Dart中创建对象最普通一种构造函数，而且**主构造函数只能有一个，如果没有指定主构造函数，那么会默认自动分配一个默认无参的主构造函数**。此外**dart中构造函数不支持重载**

```dart
class Person {
    var name;
    //隐藏了默认的无参构造函数Person();
}
//等价于:
class Person {
    var name;
    Person();//一般把与类名相同的函数称为主构造函数
}
//等价于
class Person {
    var name;
    Person(){}
}

class Person {
    final String name;
    final int age;
    Person(this.name, this.age);//显式声明有参主构造函数
    Person();//编译异常，注意: dart不支持同时重载多个构造函数。
}
```

**构造函数初始化列表** **`:`**

```dart
class Point3D extends Point {
    double z;
    Point3D(a, b, c): z = c / 2, super(a, b);//初始化列表，多个初始化步骤用逗号分隔；先初始化z ,然后执行super(a, b)调用父类的构造函数
}
//等价于
class Point3D extends Point {
    double z;
    Point3D(a, b, c): z = c / 2;//如果初始化列表没有调用父类构造函数，
    //那么就会存在一个隐含的父类构造函数super调用将会默认添加到初始化列表的尾部
}
```

初始化的顺序如下图:

![img](https://pic1.zhimg.com/80/v2-e9d9aef87e0f3a1d4fa1ceb5e2561bc4_1440w.jpg)



**初始化实例变量几种方式**

```dart
//方式一: 通过实例变量声明时直接赋默认值初始化
class Point {
    double x = 0, y = 0;
}

//方式二: 使用构造函数初始化方式
class Point {
    double x, y;
    Point(this.x, this.y);
}

//方式三: 使用初始化列表初始化
class Point {
    double x, y;
    Point(double a, double b): x = a, y = b;//:后跟初始化列表
}

//方式四: 在构造函数中初始化，注意这个和方式二还是有点不一样的。
class Point {
    double x, y;
    Point(double a, double b) {
        x = a;
        y = b;
    }
}
```

### 2、命名构造函数

通过上面主构造函数我们知道在**Dart中的构造函数是不支持重载的，实际上Dart中连基本的普通函数都不支持函数重载**。 那么问题来了，我们经常会遇到构造函数重载的场景，有时候需要指定不同的构造函数形参来创建不同的对象。所以为了解决不同参数来创建对象问题，**虽然抛弃了函数重载，但是引入命名构造函数的概念**。它可以指定任意参数列表来构建对象，只不过的是需要给构造函数指定特定的名字而已。

```dart
class Person {
  final String name;
  int age;

  Person(this.name, this.age);

  Person.withName(this.name);//通过类名.函数名形式来定义命名构造函数withName。只需要name参数就能创建对象，
  //如果没有命名构造函数，在其他语言中，我们一般使用函数重载的方式实现。
}

main () {
  var person = Person('mikyou', 18);//通过主构造函数创建对象
  var personWithName = Person.withName('mikyou');//通过命名构造函数创建对象
}
```

### 3、重定向构造函数

有时候需要将**构造函数重定向到同一个类中的另一个构造函数**，**重定向构造函数的主体为空，构造函数的调用出现在冒号(:)之后**。

```dart
class Point {
    double x, y;
    Point(this.x, this.y);
    Point.withX(double x): this(x, 0);//注意这里使用this重定向到Point(double x, double y)主构造函数中。
}
//或者
import 'dart:math';

class Point {
  double distance;

  Point.withDistance(this.distance);

  Point(double x, double y) : this.withDistance(sqrt(x * x + y * y));//注意:这里是主构造函数重定向到命名构造函数withDistance中。
}
```

### 4、factory工厂构造函数

一般来说，构造函数总是会创建一个新的实例对象。但是有时候会遇到并不是每次都需要创建新的实例，可能需要使用缓存，如果仅仅使用上面普通构造函数是很难做到的。那么这时候就需要**factory工厂构造函数**。**它使用`factory`关键字来修饰构造函数，并且可以从缓存中的返回已经创建实例或者返回一个新的实例**。在dart中任意构造函数都可以被替换成工厂方法， 它看起来和普通构造函数没什么区别，可能没有初始化列表或初始化参数，但是它**必须有一个返回对象的函数体**。

```dart
class Logger {
  //实例属性
  final String name;
  bool mute = false;

  // _cache is library-private, thanks to
  // the _ in front of its name.
  static final Map<String, Logger> _cache =
      <String, Logger>{};//类属性

  factory Logger(String name) {//使用factory关键字声明工厂构造函数，
    if (_cache.containsKey(name)) {
      return _cache[name]//返回缓存已经创建实例
    } else {
      final logger = Logger._internal(name);//缓存中找不到对应的name logger，调用_internal命名构造函数创建一个新的Logger实例
      _cache[name] = logger;//并把这个实例加入缓存中
      return logger;//注意: 最后返回这个新创建的实例
    }
  }

  Logger._internal(this.name);//定义一个命名私有的构造函数_internal

  void log(String msg) {//实例方法
    if (!mute) print(msg);
  }
}
```

### 三、抽象方法、抽象类和接口

抽象方法就是**声明一个方法而不提供它的具体实现**。任何实例的方法都可以是抽象的，包括getter,setter,操作符或者普通方法。含有抽象方法的类本身就是一个抽象类，抽象类的声明使用关键字 **`abstract`**.

```dart
abstract class Person {//abstract声明抽象类
  String name();//抽象普通方法
  get age;//抽象getter
}

class Student extends Person {//使用extends继承
  @override
  String name() {
    // TODO: implement name
    return null;
  }

  @override
  // TODO: implement age
  get age => null;
}
```

在Dart中并没有像其他语言一样有个 `interface`的关键字修饰。因为**Dart中每个类都默认隐含地定义了一个接口**。

```dart
abstract class Speaking {//虽然定义的是抽象类，但是隐含地定义接口Speaking
  String speak();
}

abstract class Writing {//虽然定义的是抽象类，但是隐含地定义接口Writing
  String write();
}

class Student implements Speaking, Writing {//使用implements关键字实现接口
  @override
  String speak() {//重写speak方法
    // TODO: implement speak
    return null;
  }

  @override
  String write() {//重写write方法
    // TODO: implement write
    return null;
  }
}
```

### 四、类函数

类函数顾名思义就是**类的函数，它不属于任何一个实例，所以它也就不能被继承**。类函数使用 **`static`** 关键字修饰，调用时可以直接使用类名.函数名的方式调用。

```dart
class Point {
    double x,y;
    Point(this.x, this.y);
    static double distance(Point p1, Point p2) {//使用static关键字，定义类函数。
        var dx = p1.x - p2.x;
        var dy = p1.y - p2.y;
        return sqrt(dx * dx + dy * dy);
    }
}
main() {
    var point1 = Point(2, 3);
    var point2 = Point(3, 4);
    print('the distance is ${Point.distance(point1, point2)}');//使用Point.distance => 类名.函数名方式调用
}
```

### 总结

到这里有关dart中面向对象基础部分已经介绍完毕，这篇文章主要介绍了dart中常用的构造函数以及一些面向对象基础知识，下一篇我们将继续dart中面向对象一些高级的东西，比如面向对象的继承和mixin. 欢迎关注~~~