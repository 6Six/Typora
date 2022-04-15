# [objc_msgSend函数的底层实现](https://chy305chy.github.io/2018/08/14/%E4%BB%8E%E4%B8%80%E4%B8%AABUG%E8%B0%88%E8%B5%B7%EF%BC%8C%E5%89%96%E6%9E%90objc-msgSend%E5%87%BD%E6%95%B0%E7%9A%84%E5%BA%95%E5%B1%82%E5%AE%9E%E7%8E%B0/)



### 一、引出问题

---

前段时间开发FLEX+Relative库（使用Category和Runtime实现将[FLEX](https://github.com/Flipboard/FLEX)库扩展出可以查看UIView相对间距的功能）。开发完成后遇到了一个奇怪的BUG，模拟器与真机调试时行为不一致，模拟器上可以正常实现预期功能，但是在真机上却出现问题：程序展示的视图与实际选中的视图不一致。先看问题代码：

```
- (void)updateRelativeViewsForSelectionPoint:(CGPoint)selectionPointInWindow {
    [self removeAndClearRelativeLines];
    UIView *selectedView = objc_msgSend(self, @selector(viewForSelectionAtPoint:), selectionPointInWindow);
    if ([self.relativeViews containsObject:selectedView]) {
        [self.relativeViews removeObject:selectedView];
    } else {
        [self.relativeViews addObject:selectedView];
    }
    if (self.relativeViews.count > 2) {
        [self.relativeViews removeObjectAtIndex:0];
    }
    [self updateRelativeDimensionLines];
    [self.view bringSubviewToFront:(UIView *)[self valueForKey:@"explorerToolbar"]];
    [self performSelector:@selector(updateButtonStates)];
}
```

通过调试，定位到问题出现在objc_msgSend函数调用这一行，这行代码的作用是调用FLEX库的viewForSelectionAtPoint:方法返回用户点击的UIView，入参为用户点击的坐标。调试过程中发现，通过objc_msgSend函数调用时，传入viewForSelectionAtPoint:方法的坐标发生了变化（x和y值发生了互换，在模拟器上正常）。



我们知道objc_msgSend是OC中的核心函数，所有的方法调用最终都会转化成objc_msgSend函数调用，Apple为了优化其性能，该函数内部使用汇编语言实现，而且不同平台对应不同的汇编文件，你可以在这里[objc_msgSend汇编源码](https://opensource.apple.com/source/objc4/objc4-723/runtime/Messengers.subproj/)查阅相关源代码。objc_msgSend中使用了cache，而且为了实现极致的性能优化，该函数使用了ldp指令、编译内存屏障（Compile Memory Barrier）、内存垃圾回收等技术代替锁来解决多线程环境下的读写竞争和死锁问题。



通过查阅资料发现，objc_msgSend函数与正常的C函数不同，调用时要准确地指定其返回值、入参等的类型：

> This unusual casting situation arises because objc_msgSend is not intended to be called like a normal C function. It is (and must be) implemented in assembly, and just jumps to a target C function after fiddling with a few registers. In particular, there is no consistent way to refer to any argument past the first two from within objc_msgSend. Another case where just calling objc_msgSend straight wouldn’t work is a method that returns an NSRect, say, because objc_msgSend is not used in that case, objc_msgSend_stret is. In the underlying C function for a method that returns an NSRect, the first argument is actually a pointer to an out value NSRect, and the function itself actually returns void. You must match this convention when calling because it’s what the called method will assume. Further, the circumstances in which objc_msgSend_stret is used differ between architectures. There is also an objc_msgSend_fpret, which should be used for methods that return certain floating point types on certain architectures.



因此，修改objc_msgSend函数调用后，问题得到解决：

```
UIView *selectedView = ((UIView* (*)(id, SEL, CGPoint p))objc_msgSend)(self, @selector(viewForSelectionAtPoint:), selectionPointInWindow);
```



### 二、问题定位

---

### 2.1 预备知识

开始之前，我们先了解一下ARM架构下程序的内存分配和汇编指令的相关知识。

##### 2.1.1 内存分配与管理





### 三、刨根问底--objc_msgSend函数分析

---









