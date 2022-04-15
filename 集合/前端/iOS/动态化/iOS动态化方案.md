# [iOS 动态化方案](https://km.woa.com/group/39561/articles/show/433537?kmref=search&from_page=1&no=1)



| 导语 动态化：基于线上版本快速迭代开发

### 1、什么是动态化

“动态化”是最近几年移动端比较流行的技术。

它是指在不发版的前提下，就可以动态替换或新增线上功能模块，加快版本迭代速度。

### 2、动态化的原因

由于苹果的审核机制非常严格，对于线上问题只能通过发版解决。

在这个审核周期和版本覆盖周期内，不仅影响用户体验，而且可能对业务造成影响。

对开发者而言，出现线上问题，也是一件非常痛苦的事情。

所以就有了各种各样的动态化方案，有的支持跨平台，有的专注于特定平台。

今天我们就来聊下iOS平台的动态化方案。总体来说，可以归纳为三大类：

### 3、基于WebView的动态化方案

这是既古老又流行的动态化技术。

开发者使用HTML/CSS/JS开发Web页面，然后部署到服务器，最后App使用WebView加载URL来渲染页面。

这类方案的整体架构如下图所示：

![img](https://km.woa.com/gkm/api/img/cos-file-url?url=https%3A%2F%2Fkm-pro-1258638997.cos.ap-guangzhou.myqcloud.com%2Ffiles%2Fphotos%2Fcaptures%2F202008%2F1596211452_93_w846_h660.png&is_redirect=1)

为了丰富App的能力，我们可以提供一些插件（Plugins），用于访问系统或业务功能，例如拍照、定位等。

这类方案接入成本低，开发效率高，并且由系统平台提供支持，可以保证用户看到的都是最新的内容。

**但是，它最大的问题就在于性能和用户体验，最直观的感受就是页面加载慢。**

为了优化WebView的性能体验问题，业界在缓存、资源、网络等方面进行了各种优化。

具有代表的方案有：Ionic、Apache Cordova、微信小程序、QQ的VasSonic、支付宝的Nebula。

### 4、基于JavaScriptCore的动态化方案

为了优化用户体验，这类方案废弃了WebView，而是直接使用原生组件渲染页面，然后利用JavaScriptCore引擎执行业务逻辑，从而实现动态化。方案架构如下图所示：

![img](https://km.woa.com/gkm/api/img/cos-file-url?url=https%3A%2F%2Fkm-pro-1258638997.cos.ap-guangzhou.myqcloud.com%2Ffiles%2Fphotos%2Fcaptures%2F202008%2F1596340931_19_w848_h656.png&is_redirect=1)

这类方案的核心就是利用了JavaScriptScore引擎的动态执行能力。

业界具有代表的方案主要有：RN（React Native）、Weex、MXFlutter、Hippy

其中RN和Weex是直接使用平台提供的原生控件渲染页面，而MXFlutter是基于Flutter的Dart引擎渲染绘制页面。

这类方案基本上接近了原生的用户体验，比较好的解决了WebView的性能体验问题，并且也支持跨平台。

**但是，这类方案的接入成本是很高的。**

首先，这些都是大型的框架，需要投入人力和时间去研究其中的原理和业务的最佳实践方式。

其次，这些框架本身也在迭代发展，避免不了有很多坑需要去填补，后期的维护成本也是很高的。

虽然大家都在尝试这些方案，但完全基于这些方案开发App的可能比较少，实际情况是那些已经用H5实现的页面，换成这种方式来提升用户体验，App的主要功能页面还是用原生控件进行开发，至少我们动漫团队是这样的。

**所以，这类方案的收益也不大，并且只能实现App的局部动态化。**

那有没有一种成本低、收益高的动态化方案呢？继续往下看

### 5、基于runtime的动态化方案

这类方案基于runtime运行时调用方法的能力，使用JavaScriptCore作为代码的执行环境，实现App的全局动态化。

![img](https://km.woa.com/gkm/api/img/cos-file-url?url=https%3A%2F%2Fkm-pro-1258638997.cos.ap-guangzhou.myqcloud.com%2Ffiles%2Fphotos%2Fcaptures%2F202008%2F1596378391_64_w848_h652.png&is_redirect=1)

业界具有代表的方案有：JSPatch、DynamicCocoa、DynamicOC

其中，JSPatch使用JS编写热更新代码，由JavaScriptCore引擎负责解释执行，涉及到OC方法调用的部分会转发给runtime。

对于iOS开发而言，编写JS代码还是有学习成本的，所以滴滴的DynamicCocoa框架可以让开发者直接使用OC编写热更新代码，然后通过工具转换为特定格式的JS代码，再下发到App，交给JavaScriptCore解释执行。

由于JavaScriptCore引擎和Native之间的数据通信会带来一定的性能损耗，所以手Q团队重新开发了一套解释器OCSVM，用来替换JavaScriptCore，通过将OC编译为OCScript字节码，然后下发给OCSVM动态解释执行。

**这类方案接入成本低，对App安装包大小几乎没什么影响，不仅可以实现热更新，而且还能做功能插件化，减少包体积。**

但是，由于这类方案可以通过动态下发执行任意代码，存在被恶意截获修改的风险，可能给App或系统带来安全隐患。

**所以苹果禁止了这类方案。**当然，也可以通过一些黑客手段来绕开这类审核。



**最后做一个总结：**

本文介绍了iOS平台至今为止的动态化方案，并将其归纳为3类：

1、基于WebView的动态化：开发效率高、用户体验差

2、基于JavaScriptCore的动态化：接入成本高、实际收益小

3、基于runtime的动态化：功能太强大、无法上线（被拒）



### 参考文章：

1、[iOS 动态化的故事](https://blog.cnbang.net/tech/3286/)

2、[How Browsers Work](https://www.html5rocks.com/en/tutorials/internals/howbrowserswork/)

3、[深入理解JavaScriptCore](https://tech.meituan.com/2018/08/23/deep-understanding-of-jscore.html)

4、[基于JS的高性能Flutter动态化框架MXFlutter](https://juejin.im/post/5d11a4f06fb9a07ec63b21ea)

5、[JSPatch实现原理详解](https://blog.cnbang.net/tech/2808/)

6、[滴滴 iOS 动态化方案 DynamicCocoa 的诞生与起航](http://www.cocoachina.com/articles/18400)

7、[OCS:史上最疯狂的iOS动态化方案](https://www.iteye.com/news/32017)