# [NSProxy是什么？](https://juejin.cn/post/6979859480229969957)



## 介绍

在苹果 Foundation 体系下，有一个和 NSObject 平级的类，那就是 NSProxy。

来自苹果官方文档对NSProxy的定义：

```
An abstract superclass defining an API for objects that act as stand-ins for other objects or for objects that don't exist yet.
```

一个未某些对象定义了API的抽象父类，这些对象充当了一个或多个尚不存在的对象的替身。

需要注意的点：

- 抽象父类
- API
- 替身



## 抽象父类

作为抽象父类，意味着我们无法直接使用这个类，而必须创建新类来继承它，才可以使用这个类中对应的API。这一点上，`NSProxy`与常用的`NSOperation`是一致的。

看一下官方文档中`NSProxy`的概述部分。

> NSProxy implements the basic methods required of a root class, including those defined in the NSObject protocol.

`NSProxy`实现了根类要求的基本方法，包括那些定义在`NSObject`协议中的。

> However, as an abstract class it doesn’t provide an initialization method, and it raises an exception upon receiving any message it doesn’t respond to.

然而，作为一个抽象类，它没有提供一个初始化方法，并且在接收到任何它不响应的消息时会拉起一个异常。

> A concrete subclass must therefore provide an initialization or creation method and override the forwardInvocation: and methodSignatureForSelector: methods to handle messages that it doesn’t implement itself.

因此，一个具体的子类必须提供一个初始化方法，或创建方法，并且重写 `forwardInvocation:` 和 `methodSignatureForSelector:` 方法来处理它没有自己实现的消息。

> A subclass’s implementation of forwardInvocation: should do whatever is needed to process the invocation, such as forwarding the invocation over the network or loading the real object and passing it the invocation.

子类中关于 `forwardInvocation:` 的实现应做任何它需要来处理这个`invocation`的事，比如通过网络转发或加载实际的对象并传递给 `invocation`。

> methodSignatureForSelector: is required to provide argument type information for a given message; a subclass’s implementation should be able to determine the argument types for the messages it needs to forward and should construct an NSMethodSignature object accordingly.

`methodSignatureForSelector:` 被要求提供一个给定消息的类型信息；一个子类实现应该能够决定需要被转发的消息的参数类型，并且应该依此构建一个`NSMethodSignature`对象。

> See the NSDistantObject, NSInvocation, and NSMethodSignature class specifications for more information.

查看 `NSDistantObject`, `NSInvocation`, 和 `NSMethodSignature` 类的说明以获取更多信息。

## API

从`NSProxy`的概述中，我们已经基本上明了对`NSProxy`新建子类所需要做的事：

1. 提供一个初始化方法
2. 重写 `forwardInvocation:` 和 `methodSignatureForSelector:` 方法

需要提前说明的是， `forwardInvocation:` 和 `methodSignatureForSelector:` 两个方法本身就是消息转发过程中参与转发的方法，因此重写这两个方法实际上是在为被替身对象进行消息转发。

具体代码如下：

```Objective-C
#import <Foundation/Foundation.h>

@interface OCWeakProxy : NSProxy

@property (nonatomic, weak, readonly) id target;

- (instancetype)initWithTarget:(id)target;
+ (instancetype)proxyWithTarget:(id)target;

@end

================分隔线===================

#import "OCWeakProxy.h"

@interface OCWeakProxy ()

@property (nonatomic, weak, readwrite) id target;

@end

@implementation OCWeakProxy

- (instancetype)initWithTarget:(id)target {
    self.target = target;
    return self;
}

+ (instancetype)proxyWithTarget:(id)target {
    return [[OCWeakProxy alloc] initWithTarget:target];
}

- (void)forwardInvocation:(NSInvocation *)invocation {
    SEL sel = [invocation selector];
    if ([self.target respondsToSelector:sel]) {
        [invocation invokeWithTarget:self.target];
    }
}

- (NSMethodSignature *)methodSignatureForSelector:(SEL)sel {
    return [self.target methodSignatureForSelector:sel];
}

@end
```

需要说明的是，由于`NSProxy`是抽象类，其继承子类的初始化方法仍然不好写，尤其是Swift版本的，这里暂时先写了OC版本的。

## 替身

在上述代码中，通过`-initWithTarget:`和`+proxyWithTarget:`传入的target就是被替身的对象。

借用之前讨论过的Timer循环引用问题[【iOS面试题】2.Timer的循环引用问题](https://juejin.cn/post/6959461286505807903)，这里使用NSProxy的方式再实现一次。

这里只展示子页面的代码：

```Objective-C
#import "NextViewController.h"
#import "OCWeakProxy.h"

@interface NextViewController ()

@property (nonatomic, strong) NSTimer *timer;
@property (nonatomic, assign) NSInteger count;

@end

@implementation NextViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    // Do any additional setup after loading the view.
    self.title = @"Next";
    self.view.backgroundColor = [UIColor lightGrayColor];
    self.count = 0;
    UIButton *button = [[UIButton alloc] initWithFrame:CGRectMake(100, 200, 100, 50)];
    button.backgroundColor = [UIColor redColor];
    [button addTarget:self action:@selector(clickOnButton:) forControlEvents:UIControlEventTouchUpInside];
    [self.view addSubview:button];
    
}

- (void)clickOnButton:(UIButton *)sender {
    // 通过按钮点击触发定时器，并且将target设置为Proxy
    self.timer = [NSTimer scheduledTimerWithTimeInterval:1.0 target:[OCWeakProxy proxyWithTarget:self] selector:@selector(timerInvoked) userInfo:nil repeats:YES];
}

- (void)timerInvoked {
    self.count++;
    NSLog(@"current count: %ld", self.count);
}

- (void)dealloc {
    [self.timer invalidate];
    NSLog(@"NextViewController is dealloced.");
}

@end

```

运行结果如下：

![iShot2021-07-01 15.13.53.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/896ae2ec3e9247f99d10745854d34dea~tplv-k3u1fbpfcp-watermark.awebp)

解析如下：

在整个场景中，存在着这样的引用关系：

![iShot2021-07-01 15.22.45.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/8a88bcf87a0045138ba6b23fe19595af~tplv-k3u1fbpfcp-watermark.awebp)

其中Proxy指向Controller的虚线代表这是一条弱引用，与前面OCWeakProxy中用weak来修饰一致。

当timer运行起来后，会将`timerInvoked`消息先发送给proxy，由于proxy内部没有实现对应方法，便会进入消息转发流程，最终经过内部重写的`forwardInvocation:` 和 `methodSignatureForSelector:` 两个方法将消息转发给controller，并完成调用。

由于Proxy指向Controller的是弱引用，因此当Controller退出时，便会释放，销毁指向timer的引用，并在`-dealloc`方法中调用timer的`-invalidate`方法，将timer彻底释放，从RunLoop中移除。

至此，参与引用的对象全部被销毁。



## 总结

`NSProxy` 最大的价值在于其替身的作用和声明的消息转发方法，如果在子类中进行重写，则可以通过这两个方法实现**原来方法调用间对象的消息转发和隔离**，**避免** 类似`循环引用`、`高耦合`等情况的出现。

可以解决的问题：

1、 模拟多继承

```
@interface TargetProxy : NSProxy
 {
     id realObject1;
     id realObject2;
 }
 -(id)initWithTarget1:(id)t1 target:(id)t2;
 
 @end
 
 @implementation TargetProxy
 
 -(id)initWithTarget1:(id)t1 target:(id)t2
 {
       realObject1 = t1;
       realObject2 = t2;
       return self;
 }
 -(void)forwardInvocation:(NSInvocation *)invocation
 {
        id target = [realObject1 methodSignatureForSelector:invocation.selector]?realObject1:realObject2;
        [invocation invokeWithTarget:target];
 }
 -(NSMethodSignature *)methodSignatureForSelector:(SEL)sel
 {
      NSMethodSignature *signature;
      signature = [realObject1 methodSignatureForSelector:sel];
      if (signature) {
           return signature;
      }
      signature = [realObject2 methodSignatureForSelector:sel];
      return signature;
 }
 -(BOOL)respondsToSelector:(SEL)aSelector
 {
      if ([realObject1 respondsToSelector:aSelector]) {
         return YES;
      }
      if ([realObject2 respondsToSelector:aSelector]) {
         return YES;
       }
     return NO;
 }
 @end
 
 使用案例：
 NSMutableArray *array = [NSMutableArray array];
 NSMutableString *string = [NSMutableString string];
 
 id proxy = [[TargetProxy alloc]initWithTarget1:array target:string];
 [proxy appendString:@"This "];
 
 [proxy appendString:@"is "];
 [proxy addObject:string];
 [proxy appendString:@"a "];
 [proxy appendString:@"test!"];
  NSLog(@"count should be 1,it is:%ld",[proxy count]);
 if ([[proxy objectAtIndex:0] isEqualToString:@"This is a test!"]) {
     NSLog(@"Appending successful: %@",proxy);
 }else
 {
     NSLog(@"Appending failed, got: %@", proxy);
 }
     NSLog(@"Example finished without errors.");
 //TargetProxy拥有了NSSting与NSArray俩个类的方法属性


```



2、解决NSTimer无法释放的问题

如上



3、实现多个不同对象的消息分发

```
#import <Foundation/Foundation.h>
 
 @protocol TeacherProtocol <NSObject>
 - (void)beginTeachering;
 @end
 
 @interface Teacher : NSObject
 
 @end

 #import "Teacher.h"
 
 @implementation Teacher
 
 -(void)beginTeachering
 {
     NSLog(@"%s",__func__);
 }
 @end

 #import <Foundation/Foundation.h>
 
 @protocol StudentProtocol <NSObject>
  - (void)beginLearning;
 @end
 
 
 @interface Student : NSObject
 @end
 
 #import "Student.h"
 
 @implementation Student
 
 -(void)beginLearning
 {
     NSLog(@"%s",__func__);
 }
 @end

 #import <Foundation/Foundation.h>
 #import "Teacher.h"
 #import "Student.h"
 
 @interface JSDistProxy : NSProxy<TeacherProtocol,StudentProtocol>
 +(instancetype)sharedInstance;
 -(void)registerMethodWithTarget:(id)target;
 @end

 #import "JSDistProxy.h"
 #import <objc/runtime.h>
 
 @interface JSDistProxy ()
 @property(nonatomic,strong) NSMutableDictionary *selectorMapDic;
 @end
 
 
 @implementation JSDistProxy
 
 +(instancetype)sharedInstance
 {
    static JSDistProxy *proxy;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
       proxy = [JSDistProxy alloc];
       proxy.selectorMapDic = [NSMutableDictionary dictionary];
    });
     return proxy;
 }
 -(void)registerMethodWithTarget:(id)target
 {
     unsigned int count = 0;
     Method *methodList = class_copyMethodList([target class], &count);
     for (int i=0; i<count; i++) {
          Method method = methodList[i];
          SEL selector = method_getName(method);
          const char *method_name = sel_getName(selector);
         [self.selectorMapDic setValue:target forKey:[NSString stringWithUTF8String:method_name]];
     }
     free(methodList);
 }
 
 -(NSMethodSignature *)methodSignatureForSelector:(SEL)sel
 {
       NSString *methodStr = NSStringFromSelector(sel);
       if ([self.selectorMapDic.allKeys containsObject:methodStr]) {
       id target = self.selectorMapDic[methodStr];
           return [target methodSignatureForSelector:sel];
        }
       return [super methodSignatureForSelector:sel];
 }
 -(void)forwardInvocation:(NSInvocation *)invocation
 {
        NSString *methodName = NSStringFromSelector(invocation.selector);
        if ([self.selectorMapDic.allKeys containsObject:methodName]) {
            id target = self.selectorMapDic[methodName];
            [invocation invokeWithTarget:target];
        }else
        {
            [super forwardInvocation:invocation];
         }
 }

 @end
 使用案例：
 JSDistProxy *proxy = [JSDistProxy sharedInstance];
 Teacher *teacher = [Teacher new];
 Student *student = [Student new];
 [proxy registerMethodWithTarget:teacher];
 [proxy registerMethodWithTarget:student];
 
 [proxy beginLearning];
 [proxy beginTeachering];
 //实现方法的实现与声明分离，提升项目代码的可维护性，更加模块化。

```



