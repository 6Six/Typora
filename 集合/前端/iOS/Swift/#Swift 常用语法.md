# Swift 常用语法

### 声明

* let 和 var

  ```swift
  let:声明常量
  var:声明变量
  ```

* 类型标注

  ```swift
  var message : String
  ```

* 命名

  ```swift
  命名不能使用数字符号、箭头、保留的（或非法的）Unicode编码、连线与制表符，也不能以数字开头；
  推荐使用Camel-Case
  如果需要使用swift保留关键字，可以使用 **反引号`**包裹，例如：
  enum BlogStype {
  	case `default`
  	case colors
  }
  ```

### 元组

* 元组：多个值组合成一个复合值，元组内可以是任意类型

  ```swift
  let size = (width : 10, height: 10)
  print("\(size.0)")
  print("\(size.width)")
  
  // 也可以不对元素命名
  let size = (10, 10)
  ```



### 可选类型

* **用 `？` 来表示一个可选类型，例如：**

  ```swift
  // 可选类型没有明确赋值时，默认为nil
  var message = String?
  
  //要么有确切值
  message = "Hello"
  
  // 要么没有值，为nil
  message = nil
  ```

  ```swift
  swift中的nil和OC中的nil并不一样。在OC中，nil是一个指向不存在对象的指针，在swift中，nil不是指针--它是一个确切的值，用来表示值缺失。任何类型的可选状态都可以被设置为nil，不只是对象类型。
  ```

* **可选绑定**

  ```
  if let message = message {
  	...
  }
  
  guard let message = message else {
  	return
  }
  
  ```

  上述代码理解为：如果 `message` 返回的可选string包含一个值，创建一个叫做 `message` 的新常量并将可选包含的值赋给它。

* **隐式解析可选类型**

  隐式解析可选类型支持在可选类型后面加 `! `来直接获取有效值，例如:

  ```swift
  let message : String = message!
  
  ！！！ 此方法有风险，在隐式解析可选类型没有值的时候尝试取值，会触发运行时错误。
  ```

  

### 运算符

* **空合运算符** 

  * 空合运算符 `(a ?? b)` 类似于三目运算符，`a` 必须是Optional 类型，默认值 `b`  的类型必须要和 `a` 存储值类型保持一致。

    ```swift
    let message: String = message ?? "Hello"
    // 实际上等于三目运算符
    message != nil ? message! : "hello"
    ```

* **区间运算符**

  ```swift
  let array = [1, 2, 3, 4, 5]
  // 闭区间运算符，表示截取下标0~2 的数组元素
  array[0...2]
  // 半开区间运算符，表示截取下标0~1的数组元素
  array[0..<2]
  // 单侧区间运算符，表示截取开始到下标2的数组元素
  array[...2]
  // 单侧区间运算符，表示截取从下标2到结束的数组元素
  array[2...]
  ```

  除此之外，还可以通过 `...` 或者 `..<` 来连接两个字符串。

  ```swift
  // 判断是否包含大写字母，并打印
  let str = "Hello"
  let test = "A"..."Z"
  for c in str {
  	if test.contains(String(c)) {
  		print("\(c) 是大写字母")
  	}
  }
  // 打印 H是大写字母
  ```

  

* **恒等运算符 `===` 和 `==`**

  ```swift
  == 只是比较两个变量的值，不会比较它们的指针是否指向同一内存；
  == 默认比较基本类型的值，例如：int，string等；它不可以比较引用类型（reference type）或者 值类型 (value type)，除非该类实现了Equatable
  
  === 不仅比较两个变量的值，还会比较它们的指针是否指向同一内存;
  === 检查两个对象是否完全一致(它会检测对象的指针是否指向同一地址)，它只能比较引用类型(reference type)，不可以比较基本类型和值类型(type value)
  ```

* **++ 和 --**

  * swift不支持这种写法

### 闭包

​	闭包是自包含的函数代码库，可以在代码中被传递和使用。swift中的闭包 与 C 和 ObjC中的代码块（block）比较相似。

​	swift 的闭包表达式拥有简洁的风格，并鼓励在常见场景中进行语法优化，主要优化如下：

​		1、利用上下文推断参数和返回值类型

​		2、隐式返回单表达式闭包，即单表达式闭包可以省略 `return` 关键字

​		3、参数名称缩写

​		4、尾随闭包语法

​	闭包表达式语法有如下格式：

  ```swift
{
	(parameters) -> returnType in
		statements
}
  ```

- **尾随闭包**

  当函数的最后一个参数是闭包时，可以使用尾随闭包来增强函数的可读性。在使用尾随闭包时，不用写出它的参数标签：

  ```swift
  // 声明函数，参数是闭包
  func test(closure: () -> Void) {
  	...
  }
  
  // 调用方法不使用尾随闭包
  test(closure: {
  	...
  })
  
  // 调用方法使用尾随闭包
  test() {
  	...
  }
  ```

- **逃逸闭包**

  `当一个闭包作为参数传到一个函数中，但是这个闭包在函数返回之后还可能被使用，我们称该闭包从函数中逃逸。 `例如：

  ```swift
  var completions : [() -> Void] = []
  func testClosure(completion: () -> Void) {
  	completions.append(completion)
  }
  ```

  此时编译器会报错，提示你这是一个逃逸闭包，我们可以在参数名之前标注 `@escaping`，用来指明这个闭包是允许 `逃逸`  出这个函数的。

  ```swift
  var completions: [() -> Void] = []
  func testEscapingClosure(completion: @escaping () -> Void) {
  	completions.append(completion)
  }
  ```

  **注意**：将一个闭包标记为 `@escaping` 意味着你必须在闭包中显示的引用 `self`，而非逃逸闭包则不用，否则会循环引用.

- **自动闭包**

  自动闭包是一个自动创建的闭包，用于包装传递给函数作为参数的表达式。这种闭包不接受任何参数，让你能够省略闭包的花括号，用一个普通的表达式来代替显示的闭包。

  并且自动闭包让你能够延迟求值，因为直到你调用这个闭包，代码段才会被执行。要标注一个闭包时自动闭包，需要使用 `@autoclosure`.

  ```swift
  // 未使用自动闭包，需要显示用花括号说明这个参数是一个闭包
  func test(closure: () -> Bool) {
  }
  test(closure: {1 < 2})
  
  // 使用自动闭包，只需要传递表达式
  func test(closure: @autoclosure () -> String) {
  }
  tes(closure: 1< 2)
  ```

### 递归枚举

```swift
// 标记整个枚举是递归枚举
indirect enum Food {
	case beef
	case potato
	case mix(Food, Food)
}

// 仅标记存在递归的枚举成员
enum Food {
	case beef
	case potato
	indirect case mix(Food, Food)
}

推荐第二种写法，因为使用递归枚举时，编译器会插入一个间接层。仅标记枚举成员，能够减少不必要的消耗。
```



### 属性

- **存储属性**

  存储在特定类或结构体实例里的一个常量或变量。存储属性可以用`var`定义，也能用`let`定义。

  ```swift
  struct Person {
  	var name : String
  	var height : CGFloat
  }
  ```

  还可以通过 `lazy` 来标示该属性为延迟存储属性

  ```
  lazy var fileName : String = "data"
  // 懒加载非多线程安全，谨慎使用
  ```

  

- **计算属性**

  ```swift
  struct Rect {
  	var origin = CGPoint.zero
  	var size = CGSize.zero
  	var center : CGPoint {
  		get {
  			let centerX = origin.x + (size.width / 2)
  			let centerY = origin.y + (size.height / 2)
  			return Point(x: centerX, y: centerY)
  		}
  		set {
  			origin.x = newValue.x - (size.width / 2)
  			origin.y = newValue.y - (size.height / 2)
  		}
  	}
  }
  ```

  如果只希望可读而不可写时，`setter` 方法不提供即可，简写为：

  ```
  var center : CGPoint {
  	let centerX = origin.x + (size.width / 2)
  	let centerY = origin.y + (size.height / 2)
  	return Point(x: centerX, y: centerY)
  }
  ```

  

- **观察器**

  swift提供了观察属性变化的方法，每次属性被设置值时都会调用属性观察器，即使新值和旧值相同也不例外。

  ```
  var origin : CGPoint {
  	willSet {
  		print("\(newValue)")
  	}
  	didSet {
  		print("\(oldValue)")
  	}
  }
  ```

  

- **调用时序**

  调用 `number`  和  `set`  方法可以看到工作的顺序

  ```
  let b = B()
  b.number = 0
  
  // 输出
  // get
  // willSet
  // set 
  // didSet
  怎么输出的?
  ```

### unowned

在声明属性或变量时，可以在前面加上 `unowned`   表示这是一个无主引用。使用无主引用，你必须确保引用始终指向一个未销毁的实例。

和weak类似，unowned 不会牢牢保持住引用的实例。它也被用于**解决可能存在循环引用，且对象是非可选类型的场景**。

例：一个客户可能有或者没有信用卡，但是一张信用卡总是关联着一个客户

```
class Customer {
	let name : String
	var card : CreditCard?
}

class CreditCard {
	let number : UInt64
	unowned let custom : Customer		// 由于始终有值，无法使用weak
}
```

### **is 和 as**

- **is**

  `is` 在功能上类似 `isKindOfClass`，可以检查一个对象是否属于某类型或其子类型。 `is` 和原来的区别主要在于亮点，首先它不仅可以用于`class` 类型上，也可以对 `swift` 的其他像是 `struct` 或 `enum` 类型进行判断。

  ```swift
  class ClassA {}
  class ClassB : ClassA {}
  
  let obj : AnyObject = ClassB()
  
  if (obj is ClassA) {
  	print("belong to ClassA")
  }
  
  if (obj is ClassB) {
  	print("belong to ClassB")
  }
  ```

  

- **as**

  某类型的一个常量或变量可能在幕后实际上属于一个子类。当确定是这种情况，可以尝试向下转到它的子类型，用类型转换操作符（`as?` 或 `as!`）

  ```swift
  class Media {}
  class Movie : Media {}
  class Song : Media {}
  
  for item in medias {
  	if let movie = item as? Movie {
  		print("It's movie")
  	} else if let song = item as? Song {
  		print("It's song")
  	}
  }
  ```

  `as?` 返回一个试图向下转成的类型的可选值。

  `as!` 试图向下转型和强制解包转换结果结合为一个操作，只有确定向下转型一定会成功时，才使用`as!`

  **注意：** `as!` 有奔溃的风险，慎重使用

### 扩展下标

​	swift支持通过 `Extension` 为已有类型添加新下标，例：

```swift
extension Int {
	subscript(digitIndex: Int) -> Int {
		var decimalBase = 1
		for _ in 0..<digitIndex {
			decimalBase *= 10
		}
		return (self/decimalBase) % 10
	}
}
```



### mutating

结构体和枚举类型中修改 `self` 或其属性的方法必须将该实例方法标注为`mutating` ，否则无法在方法里改变自己的变量。

```swift
struct MyCar {
	var color = UIColor.blue
	mutating func changeColor() {
		color = UIColor.red
	}
}
```

由于swift的protocol 不仅可以被 `class` 类型实现，也适用于 `struc` 和 `enum`，因此我们在写给别人用的接口时需要多考虑是否使用 `mutating`  来修饰方法。



### 协议合成

有时候需要同时遵循多个协议，例如一个函数希望某个参数同时满足ProtocolA和ProtocolB，我们可以采用 `ProtocolA & ProtocolB` 这样的格式进行组合，称为 `协议合成(protocol composition)`。

```swift
func testComposition(protocols : ProtocolA & ProtocolB) {
	// ...
}
```



### selector和@objc

```swift
btn.addTarget(self, action: #selector(onClick(_:)), for .touchUpInside)

@objc func onClick(_ sender : UIButton) {
	// ....
}
```

为什么要用 `@objc` ?

因为swift中的 `#selector` 是从暴露给ObjC的代码中获取一个 selector，所以它仍然是 ObjC runtime的概念，如果selector 对应的方法只在swift中可见的话（也就是一个swift的private方法），在调用这个selector时会出现 unrecognizer selector错误。



### inout

有些时候希望在方法内部**直接**修改输入的值，这时候可以使用 `inout` 来对参数进行修改：

```
func addOne(_ variable: inout Int) {
	variable += 1
}

// 因为在函数内部更改了值，所以不需要返回了。调用也要改变为相应的形式，在前面加上 & 符号：
incrementor(&luckyNumber)
```



### 单例

在ObjC中单例一般写成：

```objective-c
@implementation MyManager
+ (id)shareManager {
	static MyManager *staticInstance = nil
	static dispatch_once_t onceToken
	
	dispatch_once(&onceToken, ^{
		staticInstance = [[self alloc] init]
	})
	
	return staticInstance
}
@end
```

在swift中变得非常简洁：

```swift
static let shareInstance = MyManager()
```



### 随机数

获取100以内的随机数：

```swift
let randomNum: Int = arc4random() % 100
```

此时编译器会提示error，因为 `arc4random()` 返回 UInt32，需要做类型转换，有时候我们可能就直接：

```swift
let randomNum : Int = Int(arc4random()) % 100
```

结果测试时在有些机型上奔溃。

**原因：**这是因为Int在32位机器上相当于Int32，64位机器上相当于Int64，表现上与ObjC中的NSInteger一致，而 `arc4random()` 始终返回 UInt32，所以在32机器上就可能越界奔溃。

最快捷的方式可以先取余之后再类型转换：

```swift
let randomNum : Int = Int(arc4random() % 100)
```



### 可变参数函数

如果想要一个可变参数的函数只需要在声明参数时在类型后面加上 `...` 就可以了。

```swift
func sum(input: Int...) -> Int {
	return input.reduce(0, combine: +)
}

print(sum(1,2,3,4,5))
```



### 可选协议

swift `procoal` 本身不允许可选项，要求所有方法都是必须得实现的。但是由于swift和ObjC可以混编，name为了方便和ObjC打交道，swift支持在 `Protocol` 中使用 `optional` 关键字作为前缀来定义可选要求，且协议和可选要求都必须带上 `@objc` 属性。

```swift
@objc protocol CounterDataSource {
	@objc optional func incrementForCount(count: Int) -> Int
	@objc optional var fixedIncrement: Int { get }
}
```

但是标记 `@objc` 特性的协议只能被继承自ObjC 类的类或者 `@objc` 类遵循，其他类以及结构体和枚举均不能遵循这种协议。这对于swift `protocol` 是一个很大的限制。

由于 `Protocol` 支持可扩展，那么我们在声明一个 `protocol` 之后再用 `extension` 的方式给出部分方法默认的实现，这样这些方法在实际的类中就是可选实现的。

```
protocol CounterDataSource {
	func incrementForCount(count: Int) -> Int
	var fixedIncrement: Int { get }
}

extension CounterDataSource {
	func incrementForCount(count: Int) -> Int {
		if count == 0 {
			return 0
		} else if count < 0 {
			return 1
		} else {
			return -1
		}
	}
}

class Counter : CounterDataSource {
	var fixedIncrement : Int = 0
}
```

### 协议扩展 

```
protocol A2 {
	func method1()
}

extension A2 {
	func method1() {
		return print("hi")
	}
	func method2() {
		return print("hi")
	}
}

struct B2 : A2 {
	func method1() {
		return print("hello")
	}
	func method2() {
		return print("hello")
	}
}

let b2 = B2()
b2.method1()
b2.method2()
```

打印结果如下：

```
hello
hello
```

结果看起来意料之中，稍作改动

```
let a2 = b2 as A2
a2.method1()
a2.method2()
```

打印结果：

```
hello
hi
```

对于 `method1` ，因为它在 `protocol` 中被定义了，因此对于一个被声明为遵守接口的类型的实现（也就是对于 `a2` ）来说，可以确定实例必然实现了 `method1` , 我们可以放心大胆地用动态派发的方式使用最终的实现（不论它是在类型中的具体实现，还是在接口扩展中的默认实现）；但是对于 `method2` 来说，我们只是在接口扩展中进行了定义，没有任何规定说它必须在最终的类型中被实现。在使用时，因为 `a2` 只是一个符合 `A2` 接口的实例，编译器对 `method2` 唯一能确定的只是在接口扩展中有一个默认实现，因此在调用时，无法确定安全，也就不会去进行动态派发，而是转而编译期间就确定的默认实现。

### 值类型和引用类型

swift 中的 `struct` 和 `enum` 定义的类型是值类型，使用 `class` 定义的为引用类型。swift中的所有内建类型都是值类型，不仅包括了传统意义像 `Int`、`Bool` 这些，甚至连 `String`、`Array`以及  `Dictionary` 都是值类型的。

- **值类型的好处**

  减少堆上内存分配和回收的次数。值类型的一个特点是在传递和赋值时进行赋值，每次赋值肯定会产生额外开销，但是在swift中这个消耗被控制在最小范围内，在没有必要赋值的时候，值类型的赋值都是不会发生的。

  ```
  var a = [1,2,3]
  var b = a
  let c = b
  
  b.append(5)  // 此时 a、c 和 b 的内存地址不再相同
  ```

  只有当值类型的内容发生改变时，值类型才会复制。

- **值类型的弊端**

  在少数情况下，显然也可能会在数组或字典中存储非常多的东西，并且还要对其中的内容进行添加或者删除。在此时，swift内建的值类型的容器类型在每次操作时都需要复制一遍，即使是存储的都是引用类型，在复制时我们还是需要存储大量的引用。

- **最佳实践**

  针对上述问题，可以通过Cocoa中的引用类型的容器类来对应这种情况，那么就是 `NSMutableArray`  和 `NSMutableDictionary`.

  所以，在使用数组和字典的最佳实践是：需要处理大量数据并频繁操作元素时，选择  `NSMutableArray`  和 `NSMutableDictionary`  更好 ，对于容器内条目小而容器本身数目多的情况，应该使用swift中的 `Array` 和 `Dictionary` .

### 获取对象类型

```swift
let str = "hello"
print("\(type(of: str))")
print("\(String(describing: object_getClass(str)))")

// String
// Optional(NSTaggedPointerString)
```

### KVO

在swift中使用**KVO**，仅限于在`NSObject`的子类中。因为**KVO**是基于**KVC（Key-Value Coding）**以及动态派发技术实现的，而这些东西都是ObjC运行时的概念。另外由于swift为了效率，默认禁用了动态派发，因此想用swift来实现**KVO**，需要做额外的工作，那就是将想要观测的对象标记为 **@objc dynamic** 。

例如，按照ObjC的使用习惯，往往会这么实现：

```swift
class Person : NSObject {
	@objc dynamic var isHealth = true
}

private var familyContext = 0
class Family : NSObject {
	override init() {
        grandpa = Person()
        super.init()
        print("爷爷身体状况: \(grandpa.isHealth ? "健康" : "不健康")")
        grandpa.addObserver(self, forKeyPath: "isHealth", options: [.new], context: &familyContext)
        DispatchQueue.main.asyncAfter(deadline: DispatchTime.now() + 3) {
            self.grandpa.isHealth = false
        }
    }

    override func observeValue(forKeyPath keyPath: String?, of object: Any?, change: [NSKeyValueChangeKey : Any]?, context: UnsafeMutableRawPointer?) {
        if let isHealth = change?[.newKey] as? Bool,
            context == &familyContext {
            print("爷爷身体状况发生了变化：\(isHealth ? "健康" : "不健康")")
        }
    }
}
```

但实际上swift4通过闭包优化了KVO的实现，可以将上述例子改为：

```swift
class Person: NSObject {
    @objc dynamic var isHealth = true
}

class Family: NSObject {

    var grandpa: Person
    var observation: NSKeyValueObservation?

    override init() {
        grandpa = Person()
        super.init()
        print("爷爷身体状况: \(grandpa.isHealth ? "健康" : "不健康")")

        observation = grandpa.observe(\.isHealth, options: .new) { (object, change) in
            if let isHealth = change.newValue {
                print("爷爷身体状况发生了变化：\(isHealth ? "健康" : "不健康")")
            }
        }

        DispatchQueue.main.asyncAfter(deadline: DispatchTime.now() + 3) {
            self.grandpa.isHealth = false
        }
    }
}
```

swift中 `observe` 会返回一个 `NSKeyValueObservation` 对象，开发者只需要管理它的声明周期，而不再需要移除观察者，因此不用担心忘记移除导致crash。

- **弊端**

  在ObjC中几乎可以没有限制的对所有满足 `KVC` 的属性进行监听，而现在我们需要属性有 `@objc dynamic` 进行修饰。但在很多情况下，监听的类属性并不满足这个条件且无法修改。目前可行的一个方案是通过属性观察器来实现一套自己的类似替代。

### lazy

前面提到过 `lazy` 可以用来标示属性延迟加载，它还可以配合像 `map` 或是 `filter` 这类接受闭包并进行运行的方法一起，让整个行为变成延时行为的。在某些情况下这么做也对性能会有不小的帮助。

例如，直接使用 `map` 时：

```swift
let data = 1...3
let result = data.map {
    (i: Int) -> Int in
    print("正在处理 \(i)")
    return i * 2
}

print("准备访问结果")
for i in result {
    print("操作后结果为 \(i)")
}

print("操作完毕")

```

其输出为：

```
// 正在处理 1
// 正在处理 2
// 正在处理 3
// 准备访问结果
// 操作后结果为 2
// 操作后结果为 4
// 操作后结果为 6
// 操作完毕

```

如果先进行一次 `lazy` 操作ode话，就能得到延时运行版本的容器：

```
let data = 1...3
let result = data.lazy.map {
    (i: Int) -> Int in
    print("正在处理 \(i)")
    return i * 2
}

print("准备访问结果")
for i in result {
    print("操作后结果为 \(i)")
}

print("操作完毕")

```

此时的运行结果为：

```
// 准备访问结果
// 正在处理 1
// 操作后结果为 2
// 正在处理 2
// 操作后结果为 4
// 正在处理 3
// 操作后结果为 6
// 操作完毕

```

对那些不需要完全运行，可能提前退出的情况，使用 `lazy` 来进行性能优化效果会非常有效。

### Log与编译符号

有时候想要将当前的文件名字和那些必要的信息作为参数一起打印出来，swift为我们准备了几个很有用的编译符号，用来处理类似这样的需求，分别如下：

| 符号      | 类型   | 描述                     |
| --------- | ------ | ------------------------ |
| #file     | String | 包含这个符号的文件的路径 |
| #line     | Int    | 符号出现处的行号         |
| #column   | Int    | 符号出现处的列           |
| #function | String | 包含这个符号的方法名字   |

这样，可以通过使用这些符号来写一个好一些的Log输出方法：

```
override func viewDidLoad() {
    super.viewDidLoad()

    detailLog(message: "嘿，这里有问题")
}

func detailLog<T>(message: T,
                  file: String = #file,
                  method: String = #function,
                  line: Int = #line) {
    #if DEBUG
    print("\((file as NSString).lastPathComponent)[\(line)], \(method): \(message)")
    #endif
}

```

### Optional Map

对数组使用 `map`  方法，这个方法能对数组中的所有元素应用某个规则，然后返回一个新的数组。

例如将数组中的所有元素乘2：

```
let nums = [1, 2, 3]
let result = nums.map{ $0 * 2 }
print("\(result)")

// 输出：[2, 4, 6]

```

但如果改成对某个`Int?` 乘2呢？期望如果这个 `Int?` 有值的话，就取出值进行乘2的操作：如果是 `nil` 的话就直接将 `nil` 赋给结果。

```
let num: Int? = 3
// let num: Int? = nil

var result: Int?
if let num = num {
    result = num * 2
}
print("\(String(describing: result))")

// num = 3时，打印Optional(6)
// num = nil时，打印nil

```

还有更优雅简洁的写法，那就是 `Optional Map`。不仅在 `Array` 或者说 `CollectionType` 里可以用 `map` ，在 `Optional` 的声明里，会发现它也有一个 `map` 方法：

```swift
/// Evaluates the given closure when this `Optional` instance is not `nil`,
/// passing the unwrapped value as a parameter.
///
/// Use the `map` method with a closure that returns a non-optional value.
/// This example performs an arithmetic operation on an
/// optional integer.
///
///     let possibleNumber: Int? = Int("42")
///     let possibleSquare = possibleNumber.map { $0 * $0 }
///     print(possibleSquare)
///     // Prints "Optional(1764)"
///
///     let noNumber: Int? = nil
///     let noSquare = noNumber.map { $0 * $0 }
///     print(noSquare)
///     // Prints "nil"
///
/// - Parameter transform: A closure that takes the unwrapped value
///   of the instance.
/// - Returns: The result of the given closure. If this instance is `nil`,
///   returns `nil`.
@inlinable public func map<U>(_ transform: (Wrapped) throws -> U) rethrows -> U?

```

如同函数说明描述，这个方法能让我们很方便地对一个Optional值做变化和操作，而不必进行手动的解包。输入会被自动用类似 Optinal Binding 的方式进行判断，如果有值，则进入 `transform`的闭包进行变换，并返回一个 `U?`；如果输入就是 `nil` 的话，则直接返回值为 `nil` 的 `U?`。

因此，上面的例子可以改为：

```swift
let num: Int? = 3
// let num: Int? = nil

let result = num.map { $0 * 2 }
print("\(String(describing: result))")

// num = 3时，打印Optional(6)
// num = nil时，打印nil

```



### Delegate

刚开始在swift中写代理时，可能会这样写：

```
protocol MyProyocol {
    func method()
}

class MyClass: NSObject {
    weak var delegate: MyProyocol?
}

```

然后会发现编译器提示错误：`'weak' must not be applied to non-class-bound 'MyProyocol'; consider adding a protocol conformance that has a class bound`。

这是因为 Swift 的 `protocol` 是可以被除了 `class` 以外的其他类型遵守的，而对于像 `struct` 或是 `enum` 这样的类型，本身就不通过引用计数来管理内存，所以也不可能用 `weak` 这样的 ARC 的概念来进行修饰。

因此想要在 Swift 中使用 `weak delegate`，我们就需要将 protocol 限制在 class 内，例如

```
protocol MyProyocol: class {
    func method()
}

protocol MyProyocol: NSObjectProtocol {
    func method()
}

```

`class`限制协议只用在`class`中，`NSObjectProtocol`限制只用在`NSObject`中，明显`class`的范围更广，日常开发中都可以使用。



### @synchronized

在ObjC日常开发中， `@synchronized`接触会比较多，这个关键字可以用来修饰一个变量，并为其自动加上和解除互斥锁，用以保证变量在作用范围内不会被其他线程改变。

但不幸的是Swift 中它已经不存在了。其实 `@synchronized` 在幕后做的事情是调用了 `objc_sync` 中的 `objc_sync_enter` 和 `objc_sync_exit` 方法，并且加入了一些异常判断。因此，在 Swift 中，如果我们忽略掉那些异常的话，我们想要 lock 一个变量的话，可以这样写：

```
private var isResponse: Bool {
    get {
        objc_sync_enter(lockObj)
        let result = _isResponse
        objc_sync_exit(lockObj)
        return result
    }

    set {
        objc_sync_enter(lockObj)
        _isResponse = newValue
        objc_sync_exit(lockObj)
    }
}

```

### 字面量

所谓字面量，就是指像特定的数字，字符串或者是布尔值这样，能够直截了当地指出自己的类型并为变量进行赋值的值。比如在下面：

```
let aNumber = 3
let aString = "Hello"
let aBool = true

```

在开发中我们可能会遇到下面这种情况：

```
public struct Thermometer {
    var temperature: Double
    public init(temperature: Double) {
        self.temperature = temperature
    }
}

```

想要创建一个`Thermometer`对象，可以使用如下代码：

```
let t: Thermometer = Thermometer(temperature: 20.0)

```

但是实际上`Thermometer`的初始化仅仅只需要一个`Double`类型的基础数据，如果能通过字面量来赋值该多好，比如：

```
let t: Thermometer = 20.0

```

其实Swift 为我们提供了一组非常有意思的接口，用来将字面量转换为特定的类型。对于那些实现了字面量转换接口的类型，在提供字面量赋值的时候，就可以简单地按照接口方法中定义的规则“无缝对应”地通过赋值的方式将值转换为对应类型。这些接口包括了各个原生的字面量，在实际开发中我们经常可能用到的有：

- ExpressibleByNilLiteral
- ExpressibleByIntegerLiteral
- ExpressibleByFloatLiteral
- ExpressibleByBooleanLiteral
- ExpressibleByStringLiteral
- ExpressibleByArrayLiteral
- ExpressibleByDictionaryLiteral

这样，我们就可以实现刚才的设想啦：

```
extension Thermometer: ExpressibleByFloatLiteral {
    public typealias FloatLiteralType = Double

    public init(floatLiteral value: Self.FloatLiteralType) {
        self.temperature = value
    }
}

let t: Thermometer = 20.0
```

### struct与class

- **共同点**
  1. 定义属性用于存储值
  2. 定义方法用于提供功能
  3. 定义下标操作使得可以通过下标语法来访问实例所包含的值
  4. 定义构造器用于生成初始化值
  5. 通过扩展以增加默认实现的功能
  6. 实现协议以提供某种标准功能

- **类更强大**
  1. 继承允许一个类继承另一个类的特征
  2. 类型转换允许在运行时检查和解释一个类实例的类型
  3. 析构器允许一个类实例释放任何其所被分配的资源
  4. 引用计数允许对一个类的多次引用

- **两者的区别**
  1. `struct`是值类型，`class`是引用类型。
  2. `struct`有一个自动生成的成员逐一构造器，用于初始化新结构体实例中成员的属性；而`class`没有。
  3. `struct`中修改 `self` 或其属性的方法必须将该实例方法标注为 `mutating`；而`class`并不需要。
  4. `struct`不可以继承，`class`可以继承。
  5. `struct`赋值是值拷贝，拷贝的是内容；`class`是引用拷贝，拷贝的是指针。
  6. `struct`是自动线程安全的；而`class`不是。
  7. `struct`存储在`stack`中，`class`存储在`heap`中，`struct`更快。
- **如何选择？**
  - 一般的建议是使用最小的工具来完成你的目标，如果 `struct`能够完全满足你的预期要求，可以多使用`struct`。



### 柯里化（Currying）

*Currying*就是把接受多个参数的方法进行一些变形，使其更加灵活的方法。函数式的编程思想贯穿于 Swift 中，而函数的柯里化正是这门语言函数式特点的重要表现。

例如有这样的一个题目：实现一个函数，输入是任一整数，输出要返回输入的整数 + 2。一般的实现为：

```
func addTwo(_ num: Int) -> Int {
    return num + 2
}
```

如果实现+3，+4，+5呢？是否需要将上面的函数依次增加一遍？我们其实可以定义一个通用的函数，它将接受需要与输入数字相加的数，并返回一个函数：

```
func add(_ num: Int) -> (Int) -> Int {
    return { (val) in
        return val + num
    }
}

let addTwo = add(2)
let addThree = add(3)
print("\(addTwo(1))  \(addThree(1))")
```

这样我们就可以通过*Curring*来输出模版来避免写重复方法，从而达到量产相似方法的目的。



### Swift中定义常量和Objective-C中定义常量的区别

Swift中使用`let`关键字来定义常量，`let`只是标识这是一个常量，它是在runtime时确定的，此后无法再修改；ObjC中使用`const`关键字来定义常量，在编译时或者编译解析时就需要确定值。



### 不通过继承，代码复用（共享）的方式有哪些？

- 全局函数
- 扩展



### 实现一个min函数，返回两个元素较小的元素

```
func min<T : Comparable>(_ a : T , b : T) -> T {
    return a < b ? a : b
}
```

### 两个元素交换

常见的一种写法是：

```
func swap<T>(_ a: inout T, _ b: inout T) {
    let tempA = a
    a = b
    b = tempA
}

```

但如果使用多元组的话，我们可以这么写：

```
func swap<T>(_ a: inout T, _ b: inout T) {
    (a, b) = (b, a)
}
```

这样一下变得简洁起来，并且没有*显示的增加额外空间*。

```
为什么要说没有显示的增加额外空间呢？
喵神在说多元组交换时，提到了没有使用额外空间。但有一些开发者认为多元组交换将会复制两个值，导致多余内存消耗，这种说法有些道理，但蜗牛并没有找到实质性的证据，如果有同学了解，可以评论补充。
另外蜗牛在[Leetcode-交换数字](https://leetcode-cn.com/problems/swap-numbers-lcci/)中测试了两种写法的执行耗时和内存消耗，基本上多元组交换执行速度上要优于普通交换，但前者的内存消耗要更高些，感兴趣的同学可以试试。
```

### map与flatmap

都会对数组中的每一个元素调用一次闭包函数，并返回该元素所映射的值，最终返回一个新数组。但`flatmap`更进一步，多做了一些事情：

1. 返回的结果中会去除`nil`，并且会解包`Optional`类型。
2. 会将N维数组变成1维数组返回。

### defer

`defer` 所声明的 block 会在当前代码执行退出后被调用。正因为它提供了一种延时调用的方式，所以一般会被用来做资源释放或者销毁，这在某个函数有多个返回出口的时候特别有用。

```
func testDefer() {
    print("开始持有资源")
    defer {
        print("结束持有资源")
    }

    print("程序运行ing")
}

// 开始持有资源
// 程序运行ing
// 结束持有资源
```

使用`defer`会方便的将前后必要逻辑放在一起，增强可读性和维护，但是不正确的使用也会导致问题。例如上面的例子，在持有之前先判断是否资源已经被其他持有：

```
func testDefer(isLock: Bool) {
    if !isLock {
        print("开始持有资源")
        defer {
            print("结束持有资源")
        }
    }

    print("程序运行ing")
}

// 开始持有资源
// 结束持有资源
// 程序运行ing
```

我们要注意到`defer`的作用域不是整个函数，而是当前的`scope`。那如果有多份`defer`呢？

```
func testDefer() {
    print("开始持有资源")

    defer {
        print("结束持有资源A")
    }

    defer {
        print("结束持有资源B")
    }

    print("程序运行ing")
}

// 开始持有资源
// 程序运行ing
// 结束持有资源B
// 结束持有资源A

```

当有多个`defer`时，后加入的先执行，可以猜测Swift使用了`stack`来管理`defer`。

### String和NSString的关系和区别

`String`是Swift类型，`NSString`是`Foundation`中的类，两者可以无缝转换。`String`是值类型，`NSString`是引用类型，前者更切合字符串的 "不变" 这一特性，并且值类型是自动多线程安全的，在使用上性能也有提升。

除非需要一些`NSString`特有的方法，否则使用`String`即可。

### 怎么获取一个String的长度？

仅仅是获取字符串的字符个数，可以使用`count`直接获取：

```
let str = "Hello你好"
print("\(str.count)") // 7

```

如果想获取字符串占用的字节数，可以根据具体的编码环境来获取：

```
print("\(str.lengthOfBytes(using: .utf8))")    // 11
print("\(str.lengthOfBytes(using: .unicode))") // 14
```

### [1, 2, 3].map{$0 * 2} 都用了哪些语法糖

1. [1, 2, 3]使用了字面量初始化，`Array`实现了`ExpressibleByArrayLiteral`协议。
2. 使用了尾随闭包。
3. 未显式声明参数列表和返回值，使用了闭包类型的自动推断。
4. 闭包只有一句代码时，可省略`return`，自动将这一句的结果作为返回值。
5. `$0`在未显式声明参数列表时，代表第一个参数，以此类推。

### 下面的代码能否正常运行？结果是什么？

```
var mutableArray = [1,2,3]
for i in mutableArray {
    mutableArray.removeAll()
    print("\(i)")
}
print(mutableArray)
```

可以正常运行，结果如下：

```
1
2
3
[]

```

为什么会调用三次？

因为`Array`是个值类型，它是写时赋值的，循环中`mutableArray`值一旦改变，`for in`上的`mutableArray`会产生拷贝，后者的值仍然是`[1, 2, 3]`，因此会循环三次。

## end







