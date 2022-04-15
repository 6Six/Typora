# [iOS Tagged Pointer](https://juejin.cn/post/6844903475562676238)



### Tagged Pointer介绍

---

TaggedPointer特点：

- TaggedPointer专门用来存储小的对象，列入NSNumber和NSDate；
- TaggedPointer指针的值不再是地址，而是真正的值。所以，实际上它不再是一个对象，它只是一个披着对象皮的普通变量而已。所以，它的内存并不存储在堆中，也不需要malloc和free；
- 在内存读取上有着3倍的效率，创建比以前快106倍；



##### 为什么要引入TaggedPointer

iPhone5s 采用64位处理器。
对于64位程序，我们的数据类型的长度是跟CPU的长度有关的。



![img](https://lc-gold-cdn.xitu.io/62208418a302db51f48a.png?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)



这样就导致了 一些对象占用的内存会翻倍。

同时 维护程序中的对象需要 分配内存，维护引用计数，管理生命周期，使用对象给程序的运行增加了负担。



##### Tagged Pointer

为了改进上面提到的内存占用和效率问题，苹果提出了Tagged Pointer对象。由于NSNumber、NSDate一类的变量本身的值需要占用的内存大小常常不需要8个字节，拿整数来说，4个字节所能表示的有符号整数就可以达到20多亿（注：2^31=2147483648，另外1位作为符号位)，对于绝大多数情况都是可以处理的。

我们可以将一个对象的指针拆成两部分，一部分直接保存数据，另一部分作为特殊标记，表示这是一个特别的指针，不指向任何一个地址。所以，引入了Tagged Pointer对象之后，64位CPU下NSNumber的内存图变成了以下这样：
**Tagged Pointer**



![img](https://lc-gold-cdn.xitu.io/60aee6e17188447638c8.png?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)



### 测试

```swift
#import 

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        // insert code here...
        NSNumber *number1 = @1;
        NSNumber *number2 = @2;
        NSNumber *number3 = @3;
        NSNumber *numberFFFF = @(0xFFFF);

        NSNumber *numberLager = @(MAXFLOAT);

        NSLog(@"number1 pointer is %p", number1);
        NSLog(@"number2 pointer is %p", number2);
        NSLog(@"number3 pointer is %p", number3);
        NSLog(@"numberLager pointer is %p", numberLager);

        /*
         2017-03-10 12:07:50.731726 TaggedPoint[1690:50438] number1 pointer is 0x127
         2017-03-10 12:07:50.731992 TaggedPoint[1690:50438] number2 pointer is 0x227
         2017-03-10 12:07:50.732011 TaggedPoint[1690:50438] number3 pointer is 0x327
         2017-03-10 12:07:50.732043 TaggedPoint[1690:50438] numberLager pointer is 0x1002006a0
         */


    }
    return 0;
}
```

以 0x127 为例 去掉 tag27(假设27为标记) 0x1 就是number 的值。
0x227
0x327
都有这种规律

numberLager 存储的值为MAXFloat 显然超过了tagged pointer 可以存储的范围。
所以打印的地址是单纯的指针地址，指向存储numberLager的内存地址。







### 对于isa指针的影响

---

因为tagged pointer 不是一个真正的对象，如果使用isa指针在编译时会报错。
如图:



![img](https://lc-gold-cdn.xitu.io/cfc5d8fd7c6f8f427b37.png?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)


提示我们改为object_getClass()
object_getClass()中做了相应的处理



由于object_getClass()没有对应的实现，只能从其他地方窥探一二
**objc-weak.mm**

```swift
weak_read_no_lock(weak_table_t *weak_table, id *referrer_id) 
{
    objc_object **referrer = (objc_object **)referrer_id;
    objc_object *referent = *referrer;
    if (referent->isTaggedPointer()) return (id)referent;
    //...
}

inline bool 
objc_object::isTaggedPointer() 
{
#if SUPPORT_TAGGED_POINTERS
    return ((uintptr_t)this & TAG_MASK);
#else
    return false;
#endif
}
```

**这里取对象的值做了一些判断**
如果是tagged pointer , 对象的值就是指针
如果非tagged pointer , 对象的值是指针指向的内存区域中的值























