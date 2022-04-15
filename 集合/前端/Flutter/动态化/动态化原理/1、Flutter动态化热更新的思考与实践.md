# [Flutter动态化热更新的思考与实践](https://juejin.cn/post/6844904116985004046)

[Flutter 动态化热更新的思考与实践（二）----Dart 代码转换AST](https://juejin.cn/post/6844904121300959246)

[Flutter 动态化热更新的思考与实践（三）---- 解析AST之Runtime](https://juejin.cn/post/6844904134970179592)

[Flutter 动态化热更新的思考与实践（四）---- 解析AST之Widget](https://juejin.cn/post/6844904135251197966)

[Flutter 动态化热更新的思考与实践（五）---- 调用AST动态化的代码](https://juejin.cn/post/6855129008083271694)



Flutter 刚出现在大家视野里的时候，首先的反应是否有动态化热更新的支持，不过目前Flutter的动态化热更新只限于调试Debug的阶段，在生产打包时是不支持这一个特性的，这主要与Flutter的编译模式有关。在Debug调试阶段，Flutter是以JIT（即时编译）模式运行，而生产打包后，是以AOT（事前编译）运行。

## JIT 与 AOT

JIT典型的例子就是浏览器的V8引擎，可以动态执行Javascript代码，Flutter的JIT与V8运行模式相同，在这种模式下，可以实现和Web前端一样“所见即所得”的开发方式，大幅提高开发的效率，缺点也同样很明显，由于要支持动态化，需要额外运行很多代码，导致运行性能严重下降，所以这种模式也仅限于开发阶段使用。那么另外AOT模式，这种模式与C/C++类似，运行前需要编译成二进制代码，优点显而易见，运行效率很高，缺点就是不支持动态化，每次代码改动都需要重新编译才能运行。

## Flutter动态化的方案思路

了解了Flutter的JIT和AOT的编译模式，就明白想要在生产环境下支持动态化基本不太可行了，除非后面Flutter的JIT引擎优化的足够好，可以用到生产环境中。不过在那之前，我们想要实现这一特性的话，就得另辟蹊径了。

首先在网上搜索了相关资料做了一番调研，目前支持动态化的移动端方案有HyBrid和ReactNative，不过两种技术使用的开发语言都是Javascript，也是利用了Javascript的动态化特性，前文已经说过Flutter生产环境下是以AOT方式运行的，所以HyBrid和RN的方案对我们不适合。

另外一个方案是在Flutter Android 上实现的热更新，是通过替换AOT编译后二进制文件实现的，从原理上这个方案可以满足我们的需求，不过还是有两个不足的地方：

1. 该方案可行的前提是Flutter的AOT编译文件可读写，其中有一个风险，如果后期Google限制了AOT编译文件的权限，无法进行Hack替换的话，那这个方案就彻底作废掉了。出于安全性考虑，这个限制很可能会出现。
2. 我们选择Flutter的一个重要原因是支持多终端运行，Android & iOS，后面还会接着支持Windows、MacOS等，如果这个方案只是运行在Android上的话，对我们来说价值不是很大。

偶然看到闲鱼技术团队的一篇文章，提到了第三种方案[《**Flutter动态化的方案对比及最佳实现**》](https://link.juejin.cn?target=https%3A%2F%2Fwww.yuque.com%2Fxytech%2Fflutter%2Femdguh)，思路是利用Dart语言声明式编程特性，以及Flutter UI的构建方式，通过将模板代码编译成AST描述文件，然后在App侧运行时动态解析这个AST描述文件，构造成Widget对象，然后通过`setState()`重新渲染UI。通过比较，这个方案目前是最合适的，没有平台差异性，各平台可通用，并且没有改变AOT的编译模式，只是在AOT之上实现了一个模拟JIT的轻量级动态执行引擎。



![image-20200406231630534](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2020/4/7/1715369fa7d9917e~tplv-t2oaga2asx-watermark.awebp)



在我们的Flutter业务代码中将同时包含Flutter AOT和AST Runtime 的部分， 其实AST Runtime 方案也不是十全十美，首先要考虑性能，我们要解析的AST不能太复杂，其次该方案有适用的场景，使用前需要根据公司产品的情况整理好标准封装的组件，这样直接在AST中进行定义，也可以减少AST解析的压力，不用过多考虑多种布局组合的情况，也可以减少我们编写DSL模板代码的成本。

## 如何实现

理论上我们分析AST Runtime的方案在技术和场景上行得通，接下来就要考虑如何实现这个方案，把方案进行拆解，我们需要完成以下4个阶段的任务：

1. 实现DSL模板代码编译为AST描述文件
2. 实现App侧AST Runtime轻量级引擎
3. 实现AST 管理后台，存储AST描述文件
4. 实现App侧AST 缓存管理，并设计与后台的同步策略

重点在于第1、2两个阶段，如何将DSL模版与AST Runtime 结合是个难点，对此，我们需要进一步分析我们的产品，以我们公司产品为例，我们的产品叫**新核云**，是专注于制造业行业的MES&ERP系统，App端的主要功能就是和后端各种Api的交互，也是ToB类App显著特点。这样App主要的功能形态就简单多了，我们之前在架构Flutter 的时候采用官方推荐的BLoC模式，将UI与业务解耦开，对于AST动态化，我们遵循同样的思路：



![image-20200406233924292](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2020/4/7/1715369faa9f7e7b~tplv-t2oaga2asx-watermark.awebp)



这样我们可以进一步区分要完成的工作：

- AST Widget 负责解析AST并构造Widget
- AST BLoC 则动态解析业务逻辑

将上图进一步细化：



![image-20200406234856010](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2020/4/7/1715369fad720500~tplv-t2oaga2asx-watermark.awebp)



再上图中，我们能看到AST Widget 真正与之交互的是AST Runtime，也就是说，对于业务逻辑这部分是动态执行的，需要通过AST Runtime开放的接口来动态执行相应的业务逻辑。

最后通过流程图再把我们的思路梳理一下：



![image-20200407003536898](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2020/4/7/1715369fadb2f054~tplv-t2oaga2asx-watermark.awebp)



整体的动态化方案思路大致就是这样子，先写到这里，接下来将逐步更新每个阶段具体的技术实现。

