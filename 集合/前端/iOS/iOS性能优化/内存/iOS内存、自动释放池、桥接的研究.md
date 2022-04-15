# [iOS内存、自动释放池、桥接](https://www.quanguanzhou.top/post/42800.html)

## 一：简述内存的管理

> 内存管理最重要的就是`谁创建谁释放的原则`，基本上可以解决我们90%以上（笔者凭借经验猜测的数字，不要太较真）的问题，但是有时候，系统做优化，会不遵循这个原则。

下面我们通过一个例子来介绍：

### 1. 首先我们定义一个`Mark`类

```
@interface Mark : NSObject
+ (Mark *)newMark;
+ (Mark *)createMark;
+ (Mark *)getMark;
@end

@implementation Mark
+ (Mark *)newMark {
    return [[Mark alloc] init];
}
+ (Mark *)createMark {
    return [[Mark alloc] init];
}
+ (Mark *)getMark {
    return [[Mark alloc] init];
}
@end
```

### 2.在控制器中调用代码

```
- (void)viewDidLoad {
    [super viewDidLoad];
    // Do any additional setup after loading the view.
    
    Mark * mark = [Mark createMark];
    Mark * mark2= [Mark getMark];
    Mark * mark3= [Mark newMark];
    
    self.mark = mark;
    self.mark2 = mark2;
    self.mark3 = mark3;
}
```

### 3. 我们进入`Hopper`来查看这些代码。

> 在 `[Mark createMark]` 和 `[Mark getMark]`的内部其实 会调用 `autoreleaseReturnValue`来放入到自动释放池中。 `[Mark newMark]`会自动返回，给持有的人

```
void * +[Mark newMark](void * self, void * _cmd) {
    rax = [Mark alloc];
    rax = [rax init];
    return rax;
}

void * +[Mark createMark](void * self, void * _cmd) {
    rax = [[Mark alloc] init];
    rax = [rax autorelease];
    return rax;
}

void * +[Mark getMark](void * self, void * _cmd) {
    rax = [[Mark alloc] init];
    rax = [rax autorelease];
    return rax;
}
```

在控制器中，我们可以知道

> 对于 `[[Mark createMark] retain]` 和 `[[Mark getMark] retain]` 其实伪代码有些`错误`，我们通过查看汇编代码可以知道会有个 `imp___stubs__objc_retainAutoreleasedReturnValue` ,其实就是把`autoRelease` 和 `retain` 合并起来，这样就不用再注入到`自动释放池中`.

```
void -[AutoReleaseVC viewDidLoad](void * self, void * _cmd) {
    rax = var_20;
    [[rax super] viewDidLoad];
    var_28 = [[Mark createMark] retain];
    var_30 = [[Mark getMark] retain];
    var_38 = [Mark newMark];
    [self setMark:var_28];
    [self setMark2:var_30];
    [self setMark3:var_38];
    objc_storeStrong(var_38, 0x0);
    objc_storeStrong(var_30, 0x0);
    objc_storeStrong(var_28, 0x0);
    return;
}
```

## 二：自动释放池

> 首先我们要搞清两个概念，一个是 `autoreleasepool`,另外一个是`AutoreleasePoolPage`，带着这两个问题，我们看下面例子。

### 1. `-rewrite-objc` 查看cpp代码

> 

```
- (void)testAutoPool {
    @autoreleasepool {
        NSObject * obj = [[NSObject alloc] init];
        NSLog(@"obj = %@",obj);
    }
}
```

通过命令 `xcrun -sdk iphoneos clang -rewrite-objc -F Foundation -arch arm64 Auto.m` , 我们看看 `@autoreleasepool` 到底是什么东西?

```
static void _I_AutoModel_testAutoPool(AutoModel * self, SEL _cmd) {
    /* @autoreleasepool */ { __AtAutoreleasePool __autoreleasepool; 
        NSObject * obj = ((NSObject *(*)(id, SEL))(void *)objc_msgSend)((id)((NSObject *(*)(id, SEL))(void *)objc_msgSend)((id)objc_getClass("NSObject"), sel_registerName("alloc")), sel_registerName("init"));
        NSLog((NSString *)&__NSConstantStringImpl__var_folders_zx_py2__3n12pd_dj7nkk36jzqm0000gn_T_AutoModel_b08238_mi_0,obj);
    }
}

struct __AtAutoreleasePool {
  __AtAutoreleasePool() {atautoreleasepoolobj = objc_autoreleasePoolPush();}
  ~__AtAutoreleasePool() {objc_autoreleasePoolPop(atautoreleasepoolobj);}
  void * atautoreleasepoolobj;
};
```

再通过`Hopper` 我们可以更加清楚看到

```
void -[AutoModel testAutoPool](void * self, void * _cmd) {
    var_28 = objc_autoreleasePoolPush();
    var_18 = [[NSObject alloc] init];
    NSLog(@"obj = %@", var_18);
    objc_storeStrong(var_18, 0x0);
    objc_autoreleasePoolPop(var_28);
    return;
}
```

> 其实 `@autoreleasepool{}` 内部就是 通过 `var_28 = objc_autoreleasePoolPush()` 和 `objc_autoreleasePoolPop(var_28);` 来作用的

通过查找源码`objc4-723`中的`NSObject.mm`，我们可以查看到 `AutoreleasePoolPage`,然后我们查看一下它的数据结构和一些方法

```
class AutoreleasePoolPage {
    magic_t const magic;
    id *next;
    // 说明 一个 autoreleasePool 对应一个线程
    pthread_t const thread; 
    AutoreleasePoolPage * const parent;
    AutoreleasePoolPage *child;
    uint32_t const depth;
    uint32_t hiwat;
};

void *objc_autoreleasePoolPush(void) {
    return AutoreleasePoolPage::push();
}

void objc_autoreleasePoolPop(void *ctxt) {
    AutoreleasePoolPage::pop(ctxt);
}
```

> 我们应该很清楚的知道，AutoreleasePoolPage 是一个 **双向链表**，为什么要设置多张Page，其实系统还是为了节省空间，类似我们的内存分页的思想。



![img](https://user-gold-cdn.xitu.io/2019/8/1/16c4a533faae4550?imageView2/0/w/1280/h/960/ignore-error/1)



> 通过 `autpreleasePoolPush`设置哨兵`nil` 也就是`begain`, 里面有个`next` 指针指定了下个obj的位置,也就是`end`。那么在调用`autoreleasePoolPop`的时候其实就是释放`end - begain` 的空间，同时 给这里的每个对象都发送一个 `release`方法。



![img](https://user-gold-cdn.xitu.io/2019/8/1/16c4a53addb350bd?imageView2/0/w/1280/h/960/ignore-error/1)



## 三：桥接的内存管理

桥接主要设计到下面几个方法,其实`CFBridgingRetain` 内部是调用 `__bridge_retained`的； `CFBridgingRelease` 内部也是调用 `__bridge_transfer`；

- `__bridge` : 不会改变原来的`ownership`；
- `p = (__bridge_retained void *)obj` 和 `p = (void *) CFBridgingRetain(obj)` : 把OC里面的ARC`ownership`所有权,给`CF类型的对象`；
- `(id)CFBridgingRelease(p)` 和 `(__bridge_transfer id)p`: 把CF的`ownership`所有权，交给 `OC对象`;

### 1.1 非持有 `CF对象 = (__bridge void *)OC对象`

> 通过 `__bridge` 把OC 转换成CF

```
- (void)brigeTest1 {
    void * p = 0;
    {
        id obj = [NSObject new];
        p = (__bridge void *)obj;
    }
    // 直接崩溃，因为p 不持有 obj 对象， 当 {} 结束的时候，obj被释放了， 所以导致崩溃
    NSLog(@" p = %@",p);
}
```

### 1.2 持有 `CF对象 = (__bridge_retained void *)OC对象` 或者 `CF对象 = (void *) CFBridgingRetain(OC对象)`

```
- (void)brigeTest2 {
    void * p = 0;
    {
        id obj = [NSObject new];
        p = (__bridge_retained void *)obj;
    }
    NSLog(@" p = %@",p);
    CFBridgingRelease(p);
}
- (void)brigeTest3 {
    void * p = 0;
    {
        id obj = [NSObject new];
        p = (void *) CFBridgingRetain(obj);
    }
    // 直接崩溃，因为p 不持有 obj 对象， 当 {} 结束的时候，obj被释放了， 所以导致崩溃
    NSLog(@" p = %@",p);
    CFBridgingRelease(p);
}
```

### 2.1 非持有 `id o = (__bridge id)p`

```
- (void)test {
    
    // 生成 CF p
    void * p = 0;
    id obj = [NSObject new];
    p = (void *) CFBridgingRetain(obj);
    
    // OC 对象
    
    // 不管理p的内存
    //Use __bridge to convert directly (no change in ownership)
    //使用 __bridge 不会改变 p的所有权
    id o = (__bridge id)p;
}
```

### 2.2 持有`id o1 = (id)CFBridgingRelease(p)` 和 `id o2 = (__bridge_transfer id)p`

```
- (void)test {
    
    // 生成 CF p
    void * p = 0;
    id obj = [NSObject new];
    p = (void *) CFBridgingRetain(obj);
    
    // OC 对象

    //Use CFBridgingRelease call to transfer ownership of a +1 'void *' into ARC
    // 把p的所有权转移给 oc对象持有
    id o1 = (id)CFBridgingRelease(p);
    // 和 CFBridgingRelease 等价
    id o2 = (__bridge_transfer id)p;
}
```

