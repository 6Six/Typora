# [Dart语法篇之集合的使用与源码解析(二)](https://zhuanlan.zhihu.com/p/89392018)



简述:

我们将继续Dart语法的第二篇集合，虽然集合在第一篇中已经介绍的差不多，但是在这篇文章中将会更加全面介绍有关Dart中的集合，因为之前只是介绍了[dart:core包](https://link.zhihu.com/?target=https%3A//api.dartlang.org/stable/2.6.0/dart-core/dart-core-library.html)中的List、Set、Map，实际上在dart中还提供一个非常丰富的[dart:collection包](https://link.zhihu.com/?target=https%3A//api.dartlang.org/stable/2.6.0/dart-collection/dart-collection-library.html), 看过集合源码小伙伴都知道dart:core包中的集合实际上是委托到dart:collection包中实现的，所以下面我也会从源码的角度去把两者联系起来。当然这里也只会选择几个常用的集合作为介绍。

### 一、List

在dart中的List集合是具有长度的可索引对象集合，它没有委托dart:collection包中集合实现，完全由内部自己实现。

- 初始化

\```dart

main() { //初始化一:直接使用[]形式初始化 List colorList1 = ['red', 'yellow', 'blue', 'green'];

```text
//初始化二: var + 泛型
    var colorList2 = <String> ['red', 'yellow', 'blue', 'green'];

    //初始化三: 初始化定长集合
    List<String> colorList3 = List(4);//初始化指定大小为4的集合,
    colorList3.add('deepOrange');//注意: 一旦指定了集合长度，不能再调用add方法，否则会抛出Cannot add to a fixed-length list。也容易理解因为一个定长的集合不能再扩展了。
   print(colorList3[2]);//null,此外初始化4个元素默认都是null

   //初始化四: 初始化空集合且是可变长的
   List<String> colorList4 = List();//相当于List<String> colorList4 =  []
   colorList4[2] = 'white';//这里会报错，[]=实际上就是一个运算符重载，表示修改指定index为2的元素为white，然而它长度为0所以找不到index为2元素，所以会抛出IndexOutOfRangeException
```

} ```

- 遍历

```
dart main() { List<String> colorList = ['red', 'yellow', 'blue', 'green']; //for-i遍历 for(var i = 0; i < colorList.length; i++) {//可以使用var或int print(colorList[i]); } //forEach遍历 colorList.forEach((color) => print(color));//forEach的参数为Function. =>使用了箭头函数 //for-in遍历 for(var color in colorList) { print(color); } //while+iterator迭代器遍历，类似Java中的iteator while(colorList.iterator.moveNext()) { print(colorList.iterator.current); } }
```

- 常用的函数

```
dart main() { List<String> colorList = ['red', 'yellow', 'blue', 'green']; colorList.add('white');//和Kotlin类似通过add添加一个新的元素 List<String> newColorList = ['white', 'black']; colorList.addAll(newColorList);//addAll添加批量元素 print(colorList[2]);//可以类似Kotlin一样，直接使用数组下标形式访问元素 print(colorList.length);//获取集合的长度，这个Kotlin不一样，Kotlin中使用的是size colorList.insert(1, 'black');//在集合指定index位置插入指定的元素 colorList.removeAt(2);//移除集合指定的index=2的元素，第3个元素 colorList.clear();//清除所有元素 print(colorList.sublist(1,3));//截取子集合 print(colorList.getRange(1, 3));//获取集合中某个范围元素 print(colorList.join('<--->'));//类似Kotlin中的joinToString方法，输出: red<--->yellow<--->blue<--->green print(colorList.isEmpty); print(colorList.contains('green')); }
```

- 构造函数源码分析

dart中的List有很多个构造器，一个主构造器和多个命名构造器。主构造器中有个length可选参数.

\```dart external factory List([int length]);//主构造器，传入length可选参数，默认为0

external factory List.filled(int length, E fill, {bool growable = false});//filled命名构造器，只能声明定长的数组

external factory List.from(Iterable elements, {bool growable = true});

factory List.of(Iterable elements, {bool growable = true}) => List.from(elements, growable: growable);//委托给List.from构造器来实现

external factory List.unmodifiable(Iterable elements);
\```

- exteranl关键字(插播一条内容)

**注意:** 问题来了，可能大家看到List源码的时候一脸懵逼，构造函数没有具体的实现。不知道有没有注意 到**exteranl** 关键字。**external修饰的函数具有一种实现函数声明和实现体分离的特性**。这下应该就明白了，也就是对应实现在别的地方。实际上你可以在DartSDK中的源码找到，以List举例，对应的是 **`sdk/sdk_nnbd/lib/_internal/vm/lib/array_patch.dart`**, 此外对应的external函数实现会有一个 **@patch注解** 修饰.

```dart
@patch
class List<E> {
  //对应的是List主构造函数的实现
  @patch
  factory List([int length]) native "List_new";//实际上这里是通过native层的c++数组来实现，具体可参考runtime/lib/array.cc

 //对应的是List.filled构造函数的实现,fill是需要填充元素值, 默认growable是false，默认不具有扩展功能
  @patch
  factory List.filled(int length, E fill, {bool growable: false}) {
    var result = growable ? new _GrowableList<E>(length) : new _List<E>(length);//可以看到如果是可变长，就会创建一个_GrowableList，否则就创建内部私有的_List
    if (fill != null) {//fill填充元素值不为null,就返回length长度填充值为fill的集合
      for (int i = 0; i < length; i++) {
        result[i] = fill;
      }
    }
    return result;//否则直接返回相应长度的空集合
  }

 //对应的是List.from构造函数的实现,可将Iterable的集合加入到一个新的集合中，默认growable是true,默认具备扩展功能
  @patch
  factory List.from(Iterable elements, {bool growable: true}) {
    if (elements is EfficientLengthIterable<E>) {
      int length = elements.length;
      var list = growable ? new _GrowableList<E>(length) : new _List<E>(length);//如果是可变长，就会创建一个_GrowableList，否则就创建内部私有的_List
      if (length > 0) {
        //只有在必要情况下创建iterator
        int i = 0;
        for (var element in elements) {
          list[i++] = element;
        }
      }
      return list;
    }

    //如果elements是一个Iterable<E>,就不需要为每个元素做类型测试
    //因为在一般情况下，如果elements是Iterable<E>，在开始循环之前会用单个类型测试替换其中每个元素的类型测试。但是注意下: 等等，我发现下面这段源码好像有点问题，难道是我眼神不好，if和else内部执行代码一样。    
    if (elements is Iterable<E>) {
      //创建一个_GrowableList
      List<E> list = new _GrowableList<E>(0);
      //遍历elements将每个元素重新加入到_GrowableList中
      for (E e in elements) {
        list.add(e);
      }
      //如果是可变长的直接返回这个list即可
      if (growable) return list;
      //否则调用makeListFixedLength使得集合变为定长集合,实际上调用native层的c++实现
      return makeListFixedLength(list);
    } else {
      List<E> list = new _GrowableList<E>(0);
      for (E e in elements) {
        list.add(e);
      }
      if (growable) return list;
      return makeListFixedLength(list);
    }
  }

  //对应的是List.unmodifiable构造函数的实现
  @patch
  factory List.unmodifiable(Iterable elements) {
    final result = new List<E>.from(elements, growable: false);
    //这里利用了List.from构造函数创建一个定长的集合result
    return makeFixedListUnmodifiable(result);
  }
  ...
}
```

对应的`List.from` sdk的源码解析

```dart
//sdk/lib/_internal/vm/lib/internal_patch.dart中的makeListFixedLength
@patch
List<T> makeListFixedLength<T>(List<T> growableList)
 native "Internal_makeListFixedLength";

//runtime/lib/growable_array.cc 中的Internal_makeListFixedLength
DEFINE_NATIVE_ENTRY(Internal_makeListFixedLength, 0, 1) {
 GET_NON_NULL_NATIVE_ARGUMENT(GrowableObjectArray, array,
 arguments->NativeArgAt(0));
 return Array::MakeFixedLength(array, /* unique = */ true);//调用Array::MakeFixedLength C++方法变为定长集合
}

//runtime/vm/object.cc中的Array::MakeFixedLength 返回一个RawArray
RawArray* Array::MakeFixedLength(const GrowableObjectArray& growable_array, bool unique) {

 ASSERT(!growable_array.IsNull());
 Thread* thread = Thread::Current();
 Zone* zone = thread->zone();
 intptr_t used_len = growable_array.Length();
 //拿到泛型类型参数，然后准备复制它们
 const TypeArguments& type_arguments =
 TypeArguments::Handle(growable_array.GetTypeArguments());

 //如果集合为空
 if (used_len == 0) {
 //如果type_arguments是空，那么它就是一个原生List，不带泛型类型参数的
    if (type_arguments.IsNull() && !unique) {
     //这是一个原生List(没有泛型类型参数)集合并且是非unique，直接返回空数组
         return Object::empty_array().raw();
    }
 // 根据传入List的泛型类型参数，创建一个新的空的数组
    Heap::Space space = thread->IsMutatorThread() ? Heap::kNew : Heap::kOld;//如果是MutatorThread就开辟新的内存空间否则复用旧的
    Array& array = Array::Handle(zone, Array::New(0, space));//创建一个新的空数组array
    array.SetTypeArguments(type_arguments);//设置拿到的类型参数
    return array.raw();//返回一个相同泛型参数的新数组
 }

 //如果集合不为空，取出growable_array中的data数组，且返回一个带数据新的数组array
 const Array& array = Array::Handle(zone, growable_array.data());
    ASSERT(array.IsArray());
    array.SetTypeArguments(type_arguments);//设置拿到的类型参数
    //这里主要是回收原来的growable_array,数组长度置为0，内部data数组置为空数组
    growable_array.SetLength(0);
    growable_array.SetData(Object::empty_array());
    //注意: 定长数组实现的关键点来了，会调用Truncate方法将array截断used_len长度
    array.Truncate(used_len);
    return array.raw();//最后返回array.raw()
}
```

总结一下`List.from`的源码实现，首先传入elements的`Iterate<E>`, 如果elements不带泛型参数，也就是所谓的原生集合类型，并且是非unique，直接返回空数组; 如果带泛型参数空集合，那么会创建新的空集合并带上原来泛型参数返回；如果是带泛型参数非空集合，会取出其中data数组，来创建一个新的复制原来数据的集合并带上原来泛型参数返回，最后需要截断把数组截断成原始数组长度。

- 为什么需要exteranl function

关键就是在于它能**实现声明和实现分离**，这样就**能复用同一套对外API的声明，然后对应多套多平台的实现**，如果对源码感兴趣的小伙伴就会发现相同API声明在js中也有另一套实现，这样不管是dart for web 还是dart for vm对于上层开发而言都是一套API，对于上层开发者是透明的。

### 二、Set

**dart:core**包中的**Set集合**实际上是**委托到dart:collection中的LinkedHashSet**来实现的。集合Set和列表List的区别在于 **集合中的元素是不能重复** 的。所以添加重复的元素时会返回false,表示添加不成功.

- Set初始化方式

```
dart main() { Set<String> colorSet= {'red', 'yellow', 'blue', 'green'};//直接使用{}形式初始化 var colorList = <String> {'red', 'yellow', 'blue', 'green'}; }
```

- 集合中的交、并、补集，在Kotlin并没有直接给到计算集合交、并、补的API

```
dart main() { var colorSet1 = {'red', 'yellow', 'blue', 'green'}; var colorSet2 = {'black', 'yellow', 'blue', 'green', 'white'}; print(colorSet1.intersection(colorSet2));//交集-->输出: {'yellow', 'blue', 'green'} print(colorSet1.union(colorSet2));//并集--->输出: {'black', 'red', 'yellow', 'blue', 'green', 'white'} print(colorSet1.difference(colorSet2));//补集--->输出: {'red'} }
```

- Set的遍历方式(和List一样)

```
dart main() { Set<String> colorSet = {'red', 'yellow', 'blue', 'green'}; //for-i遍历 for (var i = 0; i < colorSet.length; i++) { //可以使用var或int print(colorSet[i]); } //forEach遍历 colorSet.forEach((color) => print(color)); //forEach的参数为Function. =>使用了箭头函数 //for-in遍历 for (var color in colorSet) { print(color); } //while+iterator迭代器遍历，类似Java中的iteator while (colorSet.iterator.moveNext()) { print(colorSet.iterator.current); } }
```

- 构造函数源码分析

```
dart factory Set() = LinkedHashSet<E>; //主构造器委托到LinkedHashSet主构造器 factory Set.identity() = LinkedHashSet<E>.identity; //Set的命名构造器identity委托给LinkedHashSet的identity factory Set.from(Iterable elements) = LinkedHashSet<E>.from;//Set的命名构造器from委托给LinkedHashSet的from factory Set.of(Iterable<E> elements) = LinkedHashSet<E>.of;//Set的命名构造器of委托给LinkedHashSet的of
```

- 对应LinkedHashSet的源码分析,篇幅有限感兴趣可以去深入研究

\```dart abstract class LinkedHashSet implements Set { //LinkedHashSet主构造器声明带了三个函数类型参数作为可选参数,同样是通过exteranl实现声明和实现分离，要深入可找到对应的@Patch实现 external factory LinkedHashSet( {bool equals(E e1, E e2), int hashCode(E e), bool isValidKey(potentialKey)});

```text
//LinkedHashSet命名构造器from   
factory LinkedHashSet.from(Iterable elements) {
//内部直接创建一个LinkedHashSet对象
  LinkedHashSet<E> result = LinkedHashSet<E>();
  //并将传入elements元素遍历加入到LinkedHashSet中
  for (final element in elements) {
    result.add(element);
  }
  return result;
}

//LinkedHashSet命名构造器of，首先创建一个LinkedHashSet对象，通过级联操作直接通过addAll方法将元素加入到elements
factory LinkedHashSet.of(Iterable<E> elements) =>
    LinkedHashSet<E>()..addAll(elements);

void forEach(void action(E element));

Iterator<E> get iterator;
```

} ```

- 对应的 **`sdk/lib/_internal/vm/lib/collection_patch.dart`** 中的`@Patch LinkedHashSet`

\```dart @patch class LinkedHashSet { @patch factory LinkedHashSet( {bool equals(E e1, E e2), int hashCode(E e), bool isValidKey(potentialKey)}) { if (isValidKey == null) { if (hashCode == null) { if (equals == null) { return new _CompactLinkedHashSet(); //可选参数都为null,默认创建_CompactLinkedHashSet } hashCode = _defaultHashCode; } else { if (identical(identityHashCode, hashCode) && identical(identical, equals)) { return new _CompactLinkedIdentityHashSet();//创建_CompactLinkedIdentityHashSet } equals ??= _defaultEquals; } } else { hashCode ??= _defaultHashCode; equals ??= _defaultEquals; } return new _CompactLinkedCustomHashSet(equals, hashCode, isValidKey);//可选参数identical,默认创建_CompactLinkedCustomHashSet }

```text
@patch
factory LinkedHashSet.identity() => new _CompactLinkedIdentityHashSet<E>();
```

} ```

### 三、Map

**dart:core** 包中的 **Map集合** 实际上是 **委托到dart:collection中的LinkedHashMap** 来实现的。集合Map和Kotlin类似，key-value形式存储，并且 **Map对象的中key是不能重复的**

- Map初始化方式

```
dart main() { Map<String, int> colorMap = {'white': 0xffffffff, 'black':0xff000000};//使用{key:value}形式初始化 var colorMap = <String, int>{'white': 0xffffffff, 'black':0xff000000}; var colorMap = Map<String, int>();//创建一个空的Map集合 //实际上等价于下面代码，后面会通过源码说明 var colorMap = LinkedHashMap<String, int>(); }
```

- Map中常用的函数

```
dart main() { Map<String, int> colorMap = {'white': 0xffffffff, 'black':0xff000000}; print(colorMap.containsKey('green'));//false print(colorMap.containsValue(0xff000000));//true print(colorMap.keys.toList());//['white','black'] print(colorMap.values.toList());//[0xffffffff, 0xff000000] colorMap['white'] = 0xfffff000;//修改指定key的元素 colorMap.remove('black');//移除指定key的元素 }
```

- Map的遍历方式

```
dart main() { Map<String, int> colorMap = {'white': 0xffffffff, 'black':0xff000000}; //for-each key-value colorMap.forEach((key, value) => print('color is $key, color value is $value')); }
```

- Map.fromIterables将List集合转化成Map

```
dart main() { List<String> colorKeys = ['white', 'black']; List<int> colorValues = [0xffffffff, 0xff000000]; Map<String, int> colorMap = Map.fromIterables(colorKeys, colorValues); }
```

- 构造函数源码分析

\```dart external factory Map(); //主构造器交由外部@Patch实现, 实际上对应的@Patch实现还是委托给LinkedHashMap

factory Map.from(Map other) = LinkedHashMap.from;//Map的命名构造器from委托给LinkedHashMap的from

factory Map.of(Map other) = LinkedHashMap.of;//Map的命名构造器of委托给LinkedHashMap的of

external factory Map.unmodifiable(Map other);//unmodifiable构造器交由外部@Patch实现

factory Map.identity() = LinkedHashMap.identity;//Map的命名构造器identity交由外部@Patch实现

factory Map.fromIterable(Iterable iterable, {K key(element), V value(element)}) = LinkedHashMap.fromIterable;//Map的命名构造器fromIterable委托给LinkedHashMap的fromIterable

factory Map.fromIterables(Iterable keys, Iterable values) = LinkedHashMap.fromIterables;//Map的命名构造器fromIterables委托给LinkedHashMap的fromIterables
\```

- 对应LinkedHashMap构造函数源码分析

\```dart abstract class LinkedHashMap implements Map { //主构造器交由外部@Patch实现 external factory LinkedHashMap( {bool equals(K key1, K key2), int hashCode(K key), bool isValidKey(potentialKey)});

```text
//LinkedHashMap命名构造器identity交由外部@Patch实现
external factory LinkedHashMap.identity();

//LinkedHashMap的命名构造器from
factory LinkedHashMap.from(Map other) {
  //创建一个新的LinkedHashMap对象
  LinkedHashMap<K, V> result = LinkedHashMap<K, V>();
  //遍历other中的元素，并添加到新的LinkedHashMap对象
  other.forEach((k, v) {
    result[k] = v;
  });
  return result;
}

//LinkedHashMap的命名构造器of,创建一个新的LinkedHashMap对象，通过级联操作符调用addAll批量添加map到新的LinkedHashMap中
factory LinkedHashMap.of(Map<K, V> other) =>
    LinkedHashMap<K, V>()..addAll(other);

//LinkedHashMap的命名构造器fromIterable，传入的参数是iterable对象、key函数参数、value函数参数两个可选参数
factory LinkedHashMap.fromIterable(Iterable iterable,
    {K key(element), V value(element)}) {
  //创建新的LinkedHashMap对象，通过MapBase中的static方法_fillMapWithMappedIterable，给新的map添加元素  
  LinkedHashMap<K, V> map = LinkedHashMap<K, V>();
  MapBase._fillMapWithMappedIterable(map, iterable, key, value);
  return map;
}

//LinkedHashMap的命名构造器fromIterables
factory LinkedHashMap.fromIterables(Iterable<K> keys, Iterable<V> values) {
//创建新的LinkedHashMap对象，通过MapBase中的static方法_fillMapWithIterables，给新的map添加元素
  LinkedHashMap<K, V> map = LinkedHashMap<K, V>();
  MapBase._fillMapWithIterables(map, keys, values);
  return map;
}
```

}

//MapBase中的_fillMapWithMappedIterable
static void _fillMapWithMappedIterable( Map map, Iterable iterable, key(element), value(element)) { key ??= _id; value ??= _id;

```text
for (var element in iterable) {//遍历iterable，给map对应复制
    map[key(element)] = value(element);
  }
```

}

// MapBase中的_fillMapWithIterables static void _fillMapWithIterables(Map map, Iterable keys, Iterable values) { Iterator keyIterator = keys.iterator;//拿到keys的iterator Iterator valueIterator = values.iterator;//拿到values的iterator

```text
bool hasNextKey = keyIterator.moveNext();//是否有NextKey
  bool hasNextValue = valueIterator.moveNext();//是否有NextValue

  while (hasNextKey && hasNextValue) {//同时遍历迭代keys，values
    map[keyIterator.current] = valueIterator.current;
    hasNextKey = keyIterator.moveNext();
    hasNextValue = valueIterator.moveNext();
  }

  if (hasNextKey || hasNextValue) {//最后如果其中只要有一个为true,说明key与value的长度不一致，抛出异常
    throw ArgumentError("Iterables do not have same length.");
  }
}
```

\```

- Map的@Patch对应实现，对应 **`sdk/lib/_internal/vm/lib/map_patch.dart`** 中

\```dart @patch class Map { @patch factory Map.unmodifiable(Map other) { return new UnmodifiableMapView(new Map.from(other)); }

```text
@patch
factory Map() => new LinkedHashMap<K, V>(); //可以看到Map的创建实际上最终还是对应创建了LinkedHashMap<K, V>
```

} ```

### 四、Queue

Queue队列顾名思义先进先出的一种数据结构，在Dart对队列也做了一定的支持, 实际上**Queue**的实现是委托给**ListQueue**来实现。 Queue继承于`EfficientLengthIterable<E>`接口，然后`EfficientLengthIterable<E>`接口又继承了`Iterable<E>`.所以意味着Queue可以向List那样使用丰富的操作函数。并且由Queue派生出了 `DoubleLinkedQueue`和`ListQueue`

- 初始化

\```dart import 'dart:collection';//注意: Queue位于dart:collection包中需要导包

main() { //通过主构造器初始化 var queueColors = Queue(); queueColors.addFirst('red'); queueColors.addLast('yellow'); queueColors.add('blue'); //通过from命名构造器初始化 var queueColors2 = Queue.from(['red', 'yellow', 'blue']); //通过of命名构造器初始化 var queueColors3 = Queue.of(['red', 'yellow', 'blue']); } ```

- 常用的函数

```
dart import 'dart:collection';//注意: Queue位于dart:collection包中需要导包 main() { var queueColors = Queue() ..addFirst('red') ..addLast('yellow') ..add('blue') ..addAll(['white','black']) ..remove('black') ..clear(); }
```

- 遍历

\```dart import 'dart:collection'; //注意: Queue位于dart:collection包中需要导包

main() { Queue colorQueue = Queue.from(['red', 'yellow', 'blue', 'green']); //for-i遍历 for (var i = 0; i < colorQueue.length; i++) { //可以使用var或int print(colorQueue.elementAt(i)); //注意: 获取队列中的元素不用使用colorQueue[i], 因为Queue内部并没有去实现[]运算符重载 } //forEach遍历 colorQueue.forEach((color) => print(color)); //forEach的参数为Function. =>使用了箭头函数 //for-in遍历 for (var color in colorQueue) { print(color); } } ```

- 构造函数源码分析

\```dart factory Queue() = ListQueue;//委托给ListQueue主构造器

```text
factory Queue.from(Iterable elements) = ListQueue<E>.from;//委托给ListQueue<E>的命名构造器from

factory Queue.of(Iterable<E> elements) = ListQueue<E>.of;//委托给ListQueue<E>的命名构造器of
```

\```

- 对应的ListQueue的源码分析

\```dart class ListQueue extends ListIterable implements Queue { static const int _INITIAL_CAPACITY = 8;//默认队列的初始化容量是8 List _table; int _head; int _tail; int _modificationCount = 0;

```text
ListQueue([int? initialCapacity])
    : _head = 0,
      _tail = 0,
      _table = List<E?>(_calculateCapacity(initialCapacity));//有趣的是可以看到ListQueque内部实现是一个List<E?>集合, E?还是一个泛型类型为可空类型，但是目前dart的可空类型特性还在实验中，不过可以看到它的源码中已经用起来了。

//计算队列所需要容量大小
static int _calculateCapacity(int? initialCapacity) {
  //如果initialCapacity为null或者指定的初始化容量小于默认的容量就是用默认的容量大小
  if (initialCapacity == null || initialCapacity < _INITIAL_CAPACITY) {

        return _INITIAL_CAPACITY;
  } else if (!_isPowerOf2(initialCapacity)) {//容量大小不是2次幂
    return _nextPowerOf2(initialCapacity);//找到大小是接近number的2次幂的数
  }
  assert(_isPowerOf2(initialCapacity));//断言检查
  return initialCapacity;//最终返回initialCapacity,返回的容量大小一定是2次幂的数
}

//判断容量大小是否是2次幂
static bool _isPowerOf2(int number) => (number & (number - 1)) == 0;

//找到大小是接近number的二次幂的数
static int _nextPowerOf2(int number) {
  assert(number > 0);
  number = (number << 1) - 1;
  for (;;) {
    int nextNumber = number & (number - 1);
    if (nextNumber == 0) return number;
    number = nextNumber;
  }
}

//ListQueue的命名构造函数from
factory ListQueue.from(Iterable<dynamic> elements) {
  //判断elements 是否是List<dynamic>类型
  if (elements is List<dynamic>) {
    int length = elements.length;//取出长度
    ListQueue<E> queue = ListQueue<E>(length + 1);//创建length + 1长度的ListQueue
    assert(queue._table.length > length);//必须保证新创建的queue的长度大于传入elements的长度
    for (int i = 0; i < length; i++) {
      queue._table[i] = elements[i] as E;//然后就是给新queue中的元素赋值，注意需要强转成泛型类型E
    }
    queue._tail = length;//最终移动队列的tail尾部下标，因为可能存在实际长度大于实际元素长度
    return queue;
  } else {
    int capacity = _INITIAL_CAPACITY;
    if (elements is EfficientLengthIterable) {//如果是EfficientLengthIterable类型，就将elements长度作为初始容量不是就使用默认容量
      capacity = elements.length;
    }
    ListQueue<E> result = ListQueue<E>(capacity);
    for (final element in elements) {
      result.addLast(element as E);//通过addLast从队列尾部插入
    }
    return result;//最终返回result
  }
}

//ListQueue的命名构造函数of
factory ListQueue.of(Iterable<E> elements) =>
    ListQueue<E>()..addAll(elements); //直接创建ListQueue<E>()并通过addAll把elements加入到新的ListQueue中
...
```

} ```

### 五、LinkedList

在dart中LinkedList比较特殊，它不是一个带泛型集合，因为它泛型类型上界是`LinkedListEntry`, 内部的数据结构实现是一个双链表，链表的结点是`LinkedListEntry`的子类，且内部维护了`_next`和`_previous`指针。此外它并没有实现List接口

- 初始化

\```dart import 'dart:collection'; //注意: LinkedList位于dart:collection包中需要导包 main() { var linkedList = LinkedList>(); var prevLinkedEntry = LinkedListEntryImpl(99); var currentLinkedEntry = LinkedListEntryImpl(100); var nextLinkedEntry = LinkedListEntryImpl(101); linkedList.add(currentLinkedEntry); currentLinkedEntry.insertBefore(prevLinkedEntry);//在当前结点前插入一个新的结点 currentLinkedEntry.insertAfter(nextLinkedEntry);//在当前结点后插入一个新的结点 linkedList.forEach((entry) => print('${entry.value}')); }

//需要定义一个LinkedListEntry子类 class LinkedListEntryImpl extends LinkedListEntry> { final T value;

```text
LinkedListEntryImpl(this.value);

@override
String toString() {
  return "value is $value";
}
```

}

\```

- 常用的函数

```
dart currentLinkedEntry.insertBefore(prevLinkedEntry);//在当前结点前插入一个新的结点 currentLinkedEntry.insertAfter(nextLinkedEntry);//在当前结点后插入一个新的结点 currentLinkedEntry.previous;//获取当前结点的前一个结点 currentLinkedEntry.next;//获取当前结点的后一个结点 currentLinkedEntry.list;//获取LinkedList currentLinkedEntry.unlink();//把当前结点entry从LinkedList中删掉
```

- 遍历

```
dart //forEach迭代 linkedList.forEach((entry) => print('${entry.value}')); //for-i迭代 for (var i = 0; i < linkedList.length; i++) { print('${linkedList.elementAt(i).value}'); } //for-in迭代 for (var element in linkedList) { print('${element.value}'); }
```

### 六、HashMap

- 初始化

\```dart import 'dart:collection'; //注意: HashMap位于dart:collection包中需要导包 main() { var hashMap = HashMap();//通过HashMap主构造器初始化 hashMap['a'] = 1; hashMap['b'] = 2; hashMap['c'] = 3; var hashMap2 = HashMap.from(hashMap);//通过HashMap命名构造器from初始化 var hashMap3 = HashMap.of(hashMap);//通过HashMap命名构造器of初始化 var keys = ['a', 'b', 'c']; var values = [1, 2, 3] var hashMap4 = HashMap.fromIterables(keys, values);//通过HashMap命名构造器fromIterables初始化

```text
hashMap2.forEach((key, value) => print('key: $key  value: $value'));
```

} ```

- 常用的函数

```
dart import 'dart:collection'; //注意: HashMap位于dart:collection包中需要导包 main() { var hashMap = HashMap();//通过HashMap主构造器初始化 hashMap['a'] = 1; hashMap['b'] = 2; hashMap['c'] = 3; print(hashMap.containsKey('a'));//false print(hashMap.containsValue(2));//true print(hashMap.keys.toList());//['a','b','c'] print(hashMap.values.toList());//[1, 2, 3] hashMap['a'] = 55;//修改指定key的元素 hashMap.remove('b');//移除指定key的元素 }
```

- 遍历

`dart import 'dart:collection'; //注意: HashMap位于dart:collection包中需要导包 main() { var hashMap = HashMap();//通过HashMap主构造器初始化 hashMap['a'] = 1; hashMap['b'] = 2; hashMap['c'] = 3; //for-each key-value hashMap.forEach((key, value) => print('key is $key, value is $value')); }` * 构造函数源码分析

\```dart //主构造器交由外部@Patch实现 external factory HashMap( {bool equals(K key1, K key2), int hashCode(K key), bool isValidKey(potentialKey)});

//HashMap命名构造器identity交由外部@Patch实现 external factory HashMap.identity();

//HashMap命名构造器from factory HashMap.from(Map other) { //创建一个HashMap对象 Map result = HashMap(); //遍历other集合并把元素赋值给新的HashMap对象 other.forEach((k, v) { result[k] = v; }); return result; }

//HashMap命名构造器of，把other添加到新创建HashMap对象 factory HashMap.of(Map other) => HashMap()..addAll(other);

//HashMap命名构造器fromIterable factory HashMap.fromIterable(Iterable iterable, {K key(element), V value(element)}) { Map map = HashMap();//创建一个新的HashMap对象 MapBase._fillMapWithMappedIterable(map, iterable, key, value);//通过MapBase中的_fillMapWithMappedIterable赋值给新的HashMap对象 return map; }

//HashMap命名构造器fromIterables factory HashMap.fromIterables(Iterable keys, Iterable values) { Map map = HashMap();//创建一个新的HashMap对象 MapBase._fillMapWithIterables(map, keys, values);//通过MapBase中的_fillMapWithIterables赋值给新的HashMap对象 return map; } ```

- HashMap对应的@Patch源码实现,**`sdk/lib/_internal/vm/lib/collection_patch.dart`**

\```dart @patch class HashMap { @patch factory HashMap( {bool equals(K key1, K key2), int hashCode(K key), bool isValidKey(potentialKey)}) { if (isValidKey == null) { if (hashCode == null) { if (equals == null) { return new _HashMap();//创建私有的_HashMap对象 } hashCode = _defaultHashCode; } else { if (identical(identityHashCode, hashCode) && identical(identical, equals)) { return new _IdentityHashMap();//创建私有的_IdentityHashMap对象 } equals ??= _defaultEquals; } } else { hashCode ??= _defaultHashCode; equals ??= _defaultEquals; } return new _CustomHashMap(equals, hashCode, isValidKey);//创建私有的_CustomHashMap对象 }

```text
@patch
factory HashMap.identity() => new _IdentityHashMap<K, V>();

Set<K> _newKeySet();
```

} ```

### 七、Map、HashMap、LinkedHashMap、SplayTreeMap区别

在Dart中还有一个SplayTreeMap，它的初始化、常用的函数和遍历方式和LinkedHashMap、HashMap使用类似。但是Map、HashMap、LinkedHashMap、SplayTreeMap有什么区别呢。

- **`Map`**

Map是key-value键值对集合。在Dart中的Map中的每个条目都可以迭代的。迭代顺序取决于HashMap，LinkedHashMap或SplayTreeMap的实现。如果您使用Map构造函数创建实例，则**默认情况下会创建一个LinkedHashMap**。

- **`HashMap`**

**HashMap不保证插入顺序**。如果先插入key为A的元素，然后再插入具有key为B的另一个元素，则在遍历`Map`时，有可能先获得元素B。

- **`LinkedHashMap`**

**LinkedHashMap保证插入顺序**。根据插入顺序对存储在LinkedHashMap中的数据进行排序。如果先插入key为A的元素，然后再插入具有key为B的另一个元素，则在遍历`Map`时，总是先取的key为A的元素，然后再取的key为B的元素。

- **`SplayTreeMap`**

SplayTreeMap是一个自平衡二叉树，它允许更快地访问最近访问的元素。基本操作如插入，查找和删除可以在O（log（n））时间复杂度中完成。它通过使经常访问的元素靠近树的根来执行树的旋转。因此，如果需要更频繁地访问某些元素，则使用SplayTreeMap是一个不错的选择。但是，如果所有元素的数据访问频率几乎相同，则使用SplayTreeMap是没有用的。

### 八、命名构造函数from和of的区别以及使用建议

通过上述各个集合源码可以看到，基本上每个集合(List、Set、LinkedHashSet、LinkedHashMap、Map、HashMap等)中都有from和of命名构造函数。可能有的人有疑问了，它们有什么区别，各自的应用场景呢。其实答案从源码中就看出一点了。以List,Map中的from和of为例。

```dart
main() {
  var map = {'a': 1, 'b': 2, 'c': 3};
  var fromMap = Map.from(map); //返回类型是Map<dynamic, dynamic>
  var ofMap = Map.of(map); //返回类型是Map<String, int>

  var list = [1, 2, 3, 4];
  var fromList = List.from(list); //返回类型是List<dynamic>
  var ofList = List.of(list); //返回类型是List<int>
}
```

从上述例子可以看出List、Map中的from函数返回对应的集合泛型类型是 **`List<dynamic>`** 和 **`Map<dynamic, dynamic>`** 而of函数返回对应集合泛型类型实际类型是 **`List<int>`** 和 **`Map<String, int>`**。我们都知道dynamic是一种无法确定的类型，在编译期不检查类型，只在运行器检查类型，而具体类型是在编译期检查类型。而且从源码中可以看到 from函数往往会处理比较复杂逻辑比如需要重新遍历传入的集合然后把元素加入到新的集合中，而of函数只需要创建一个新的对象通过addAll函数批量添加传入的集合元素。

所以这里为了代码效率考虑给出建议是: **如果你传入的原有集合元素类型是确定的，请尽量使用of函数创建新的集合，否则就可以考虑使用from函数。**

### 总结

到这里我们dart语法系列第二篇就结束了，相信通过这篇文章大家对dart中的集合应该有了全面的了解，下面我们将继续研究dart和Flutter相关内容。