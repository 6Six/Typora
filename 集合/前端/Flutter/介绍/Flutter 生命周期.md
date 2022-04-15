##  Flutter 生命周期

#### Widget的生命周期：StatelessWidget和StatefulWidget

- **StatelessWidget**的生命周期只有一个build

  - build 是用来创建widget的，但因为build在每次界面刷新的时候都会调用，所以不要在build里写业务逻辑，将业务逻辑写到StatelessWidget的构造函数里；

- **StatefullWidget**的生命周期比较复杂，依次为：

  - **createState**：是StatefulWidget里创建State的方法，要创建新的StatefulWidget的时候，会立即执行createState，而且只执行一次，createState必须要实现；
  - **initState**：是StatefulWidget创建完成后调用的第一个方法，而且只执行一次，类似于Android的onCreate、iOS的viewDidLoad()，所以在view并没有渲染，但是这时StatefulWidget已经被加载到渲染树里，这时StatefulWidget的mount的值会变为true，直到dispose调用的时候才会变为false。可以在initState里做一些初始化的操作。在override initState的时候必须要调用 super.initState()。
  - **didChangeDependencies**：当StatefulWidget第一次创建的时候，didChangeDependencies方法会在initState方法之后立即调用，之后当StatefulWidget刷新的时候就不会调用了，除非StatefulWidget依赖的InheritedWidget发生变化之后，didChangeDependencies才会调用，所以didChangeDependencies有可能会被多次调用。
  - **build**：在StatefulWidget第一次创建的时候，build方法会在didChangeDependencies方法之后立即调用，另外一种会调用build方法的场景是，每当UI需要重新渲染的时候，build都会被调用，所以build会被多次调用，然后 返回要渲染的Widget。千万不要在build里做除了创建Widget之外的操作，因为这会影响UI的渲染效率。
  - **addPostFrameCallback**：**addPostFrameCallback**是StatefulWidget渲染结束的回调，只会被调用一次，之后StatefulWidget需要刷新UI也不会被调用，addPostFrameCallback的使用方法是在initState里添加回调。
  - **didUpdateWidget**：didUpdateWdiget这个生命周期一般不会用到，只有在使用key对Widget进行复用的时候才会调用。
  - **deactivate**：当要将State对象从渲染树种移除的时候，就会调用deactivate生命周期，这标志着StatefulWidget将要销毁，但是有时候State不会被销毁，而是重新插入到渲染树中。
  - **dispose**：当View不需要再显示，从渲染树中移除的时候，State就会永久的从渲染树中移除，就会调用dispose生命周期，这时候就可以在dispose里做一些取消监听、动画的操作，和initState是相反的。



#### App的生命周期 :

 想要知道Flutter App的生命周期，App在前台还是后台，需要使用到 **WidgetBindingObserver**，使用方法如下：

- State的类mix **WidgetsBindingObserver**：
  
   ```dart
   class _MyHomePageState extends State<MyHomePage> with WidgetsBindingObserver {
    ...
   }
  ```
  
   
  
- 在State的**initState**里添加监听：
  
   ```dart
   @override
     void initState() {
       super.initState();
   	 	WidgetsBinding.instance.addObserver(this);
     }
   ```
   
   
   
- 在State的**dispose**里移除监听：
  
  ```dart
  @override
  void dispose() {
    super.dispose();
    WidgetsBinding.instance.removeObserver(this);
  }
  
- 在State里override **didChangeAppLifecycleState**   

  ```
   @override
     void didChangeAppLifecycleState(AppLifecycleState state) {
       super.didChangeAppLifecycleState(state);
       if (state == AppLifecycleState.paused) {
         // went to background
       }
       if (state == AppLifecycleState.resumed) {
         // came back to foreground
    		}
     }
  ```

- **AppLifecycleState**就是App的生命周期，有：
  - **resumed**
  - **inactive**
  - **paused**
  - **suspending**



1. [MacOS下配置教程](https://flutter.cn/docs/get-started/install/macos)





















