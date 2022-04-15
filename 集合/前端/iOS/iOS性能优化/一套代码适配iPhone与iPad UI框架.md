# [如何让一套代码适配所有iOS设备尺寸](https://segmentfault.com/a/1190000037523303)



**简介：** 随着移动互联网设备和技术的发展，各种移动设备屏幕尺寸层出不穷，折叠屏、分屏、悬浮窗等等，面对越来越多样的屏幕，如果为每种尺寸单独进行适配，不仅费时费力，还会增加端侧代码的开发与维护压力。如何让一套代码适配所有尺寸变化，增强App的通用能力？阿里巴巴文娱技术 氚雨 将分享优酷APP在iOS响应式布局技术上的实践和落地。

![image.png](https://segmentfault.com/img/remote/1460000037523306)



响应式是基于同一套代码，开发一个APP能够兼容多尺寸、多终端设备的显示，能够动态调整页面的布局以及容器的布局，充分利用当前屏幕的尺寸，为用户提供更好的浏览体验，提升APP开发效率和迭代效率。

### 一 iOS布局尺寸预研

当下，iOS端的主要尺寸类型有五种：iPhone、iPad竖屏、iPad横屏、iPad浮窗、iPad分屏。通常，App是按iPhone尺寸开发的，需要适配剩余的四种iPad尺寸。

iPad横、竖屏比较常见，旋转设备即可，比较特殊的是浮窗和分屏模式。自苹果iPad iOS 9开始，用户在打开一个应用时，从最底部上滑打开Dock，即可拖拽另一个App进入浮窗模式：

![640.gif](https://segmentfault.com/img/remote/1460000037523307)

在支持分屏的iPad上拖拽到更边缘的地方即可开启分屏模式：

![640 (1).gif]().gif")

其中浮窗模式所有升级iOS 9的设备都支持，分屏模式只有最新版的硬件设备iPad mini 4、iPad Air 2及iPad Pro支持：

![image.png](https://segmentfault.com/img/remote/1460000037523309)

### 二 优酷iOS响应式方案

响应式布局的核心是设计统一的适配规则，并在屏幕尺寸发生变化时按布局规则重新布局，以适配不同屏幕尺寸，而大多数App在开发时一般只有适配iPhone的版本，在通过响应式适配更多机型时主要要解决三个方面的问题，即如何获取、更新响应式状态以进行对应的适配，如何计算在不同屏幕宽度下App内容的宽度、列数等布局参数，如何进行响应式下的数据处理以解决较难适配的组件、减少页面留白等，基于此我们开发了响应式布局SDK，负责统一管理响应式状态、处理布局逻辑、裁剪映射数据等。

![image.png](https://segmentfault.com/img/remote/1460000037523310)

#### 1 响应式App配置

App除了配置为universal版之外，要支持浮窗或分屏模式还需要进行一些配置：

（1）需要提供LaunchScreen.storyboard作为启动图，由于App支持的运行尺寸太多，不再适合用图片作为启动图。

（2）需要在info.plist中配置支持所有屏幕方向：

![image.png](https://segmentfault.com/img/remote/1460000037523311)

（3）注意不能勾选Requires full screen配置项或配置UIRequiresFullScreen为YES，如此会声明App要求全屏运行，自然表示不支持浮窗或分屏：

![image.png](https://segmentfault.com/img/remote/1460000037523313)

（4）支持分屏要求App的主Window需要使用系统UIWindow，不能继承，并且要通过init方法或initWithFrame:[UIScreen mainScreen].bounds方式初始化。

通过以上步骤开启浮窗、分屏能力后，在App内就无法再通过相关代码控制设备方向，以往通过如下代码可控制ViewController为竖屏，而支持分屏后如下方法系统不再调用，默认所有ViewController支持所有屏幕方向：

![image.png](https://segmentfault.com/img/remote/1460000037523312)

如下强制设置屏幕方向的黑方法也已失效：

![image.png](https://segmentfault.com/img/remote/1460000037523315)

这种设计的主要原因是，当一个App支持分屏后，就不再单独占用整个屏幕，当另一个App同时运行时，同一块屏幕不可能出现一个横屏、另一个竖屏。此类问题没有完美的解决方案，为了保证用户体验，支持分屏的App必须所有页面适配所有屏幕方向，这也体现了苹果对用户体验的极致追求，参见DeveloperForums中开发人员的讨论：
[https://developer.apple.com/forums/thread/19578](https://link.segmentfault.com/?enc=N8SGwlekHOwp0rnXjMtpew%3D%3D.Lg2U7zX1bEA74Br0O5C4DCS5QKV8cz2HJeGqvgcj9%2FCz8TepNa0QLTDCqesG6CTO)

#### 2 响应式SDK

**响应式状态管理**

响应式状态提供了当前是否开启响应式、响应式布局尺寸类型、当前布局window尺寸等相关状态量，响应式SDK会在屏幕尺寸变化后更新响应式状态，并通过系统通知和自定义通知机制，通知相关业务方。

```elm
// 响应式开启关闭状态
typedefNS_ENUM(NSInteger, YKRLLayoutStyle) {   
    YKRLLayoutStyleNormal =0,        // 响应式状态关闭   
    YKRLLayoutStyleResponsive =1,    // 响应式状态开启}; 
    
// 响应式屏幕尺寸类型，页面可依据此类型区分是否分屏等
typedefNS_ENUM(NSInteger, YKRLLayoutSizeType) {   
    YKRLLayoutSizeTypeS =0,      // eg. phone pad浮窗   
    YKRLLayoutSizeTypeL =1,      // pad   
    YKRLLayoutSizeTypeXL =2,     // 预留
}; 

// 响应式屏幕状态类型（一共有十种类型）
typedefNS_OPTIONS(NSUInteger, YKRLLayoutScreenType) {   
    YKRLLayoutScreenTypeUnknown = (1<<0),          //未知   
    YKRLLayoutScreenTypePortrait = (1<<1),         //竖屏全屏
    YKRLLayoutScreenTypeLandscapeLeft = (1<<2),    //横屏全屏左
    … …
};
```

响应式SDK声明了YKRLLayoutStyle、YKRLLayoutSizeType、YKRLLayoutScreenType三种枚举状态标记当前的响应式状态，分别表示响应式开启关闭状态，当前尺寸类型及具体屏幕类型，一般业务方只需要获取是否是响应式设备状态，对于在不同宽度下页面布局不一致的业务方可以通过尺寸类型状态进行区分适配，而对于需要具体知道当前屏幕状态的业务方可以通过屏幕类型获取，屏幕类型只包含当前iOS设备已支持的屏幕状态，随着设备类型的丰富，如出现折叠屏等，屏幕类型会作相应扩展。每当设备旋转或用户开启分屏时，响应式SDK都会在系统回调中更新当前响应式状态，并通知业务方响应式状态的改变。

**响应式布局规则**

优酷响应式布局规则主要包含列数适配规则、宽度适配规则等，比如多列均分组件的列数在不同屏幕宽度下是可变的，响应式SDK会根据当前的响应式状态输出合适的布局列数等，对于每一个布局规则，响应式SDK中都有相应的布局适配逻辑，响应式布局规则满足优酷App整体UI规范，业务方直接指定自己所需要的规则即可，除少数特殊规则之外，大部分布局规则都用于组件列数和组件宽度布局，此类响应式布局规则中会指定一个标准宽度，并根据组件原始布局列数和标准宽度计算出组件标准宽度，进而根据当前屏幕宽度计算出适配后的组件列数，可用如下公式表达：

> 响应式适配列数(标准屏幕宽度下组件列数) = (当前屏幕宽度÷(标准屏幕宽度÷标准屏幕宽度下组件列数×scale))

其中，scale为组件放大参数，标准屏幕宽度下组件原宽度投放到iPad上会过小，可以通过scale参数进行适当放大。

![image.png](https://segmentfault.com/img/remote/1460000037523314)

对于组件宽度适配，响应式规则会先计算标准屏幕宽度下的组件列数并进行列数适配，再通过适配后的列数计算适配宽度：

> 响应式适配宽度(标准屏幕宽度下组件宽度) = (当前屏幕宽度 - 边距间距)÷响应式适配列数(标准屏幕宽度÷标准屏幕宽度下组件宽度)

![image.png](https://segmentfault.com/img/remote/1460000037523316)

在以上公式中调整标准屏幕宽度及组件放大scale即可得到适配效果较好的通用布局规则，经过设计同学在各种设备尺寸下的调整总结，当前优酷中使用的标准屏幕宽度为440dp，scale为1.2倍，适配效果最佳。组件适配逻辑已在响应式SDK布局规则中统一实现，业务方直接调用即可，也方便设计同学对整个App的组件适配进行统一调整。

响应式SDK中YKRLCompLayoutManager类封装了相关布局逻辑，业务方也可通过YKRLCompLayoutAdapterProtocol协议二次处理，以定制响应式布局逻辑，在App统一架构中直接调用YKRLCompLayoutManager的相关接口即可获取按照响应式规则计算后的布局参数，如列数、宽度等，当监听响应式状态发生变化时重新布局即可完成响应式布局。

![image.png](https://segmentfault.com/img/remote/1460000037523317)

**响应式数据处理**
响应式数据处理包括数据映射、数据过滤、数据合并、数据补齐，数据处理逻辑两端一致，详细介绍可以参见：一个APP如何适配多个Android终端？，下面简单介绍一下iOS响应式数据映射的实现。

有些组件无法通过规则适配不同的屏幕尺寸，比如在手机上占整个屏幕宽度的组件（下图左侧带视频播放预约组件），如果采用等比放大的适配规则，在iPad端会显得过大，此类组件可以映射成相对简单的组件，以适配不同的屏幕尺寸。

![image.png](https://segmentfault.com/img/remote/1460000037523318)

优酷采用了统一抽象的数据结构，在组件映射方面比较容易实现，只需修改对应的组件标志即可。得益于统一架构的普遍推广和使用，我们在统一架构内添加了组件映射能力，方便各业务方调用，响应式SDK中提供了数据裁剪映射规则，业务方可以查询、增加相应的裁剪映射规则。对于未接入统一架构的业务方则需要业务方实现相关数据处理。

#### 3 响应式业务流程

优酷响应式业务流程两端一致，响应式布局需要进行数据处理、响应式状态管理、触发布局等工作，优酷响应式SDK会在接口返回后处理相关数据，为统一架构提供相应布局接口，监控屏幕尺寸变化并触发布局等。

![image.png](https://segmentfault.com/img/remote/1460000037523319)

#### 4 优酷响应式方案落地

iOS开发中经常采用绝对布局，而实现响应式的主要工作是将“绝对布局”修改为“相对布局”，接入工作较安卓更为繁琐。

![image.png](https://segmentfault.com/img/remote/1460000037523321)

iOS响应式可以按Window->ViewController->容器->组件的层级完成接入。

Window在配置支持分屏后会由系统自动布局，在RootViewController树中的子ViewController也会随Window自动布局，而特殊ViewController，如多tab页面的子ViewController等，未加入RootViewController树，需要手动修改为相对布局，页面可通过Autoresizing或监听响应式状态实现相对布局。

![image.png](https://segmentfault.com/img/remote/1460000037523320)

接入统一架构的页面容器由统一架构提供，统一架构容器的布局列数管理、布局宽度管理等都已接入响应式SDK，为业务方接入减少了大量工作，业务方只需指定自身所采用的布局规则即可，ViewController和容器实现相对布局后，每当屏幕尺寸变化时响应式SDK会通知容器重新布局，变换组件列数或宽度等，组件卡片只需要按容器提供的尺寸进行布局即可。

组件卡片内一般使用Frame绝对布局，需要修改为相对布局，简单的布局逻辑可以使用Autoresizing实现，方便快捷，复杂的布局可以使用AutoLayout或Masonry等自动布局框架（性能较差）实现，也可以在layoutSubviews方法中重新计算布局，业务方可以选择合适的方式实现自动布局，以减少接入成本。

对于未接入统一架构的页面则需要在本页面布局逻辑中手动接入响应式SDK相关布局接口。

![image.png](https://segmentfault.com/img/remote/1460000037523322)

落地过程中发现许多组件卡片布局时依赖了屏幕宽度，不符合响应式开发规范，导致适配响应式时工作量较大。每一层View只应依赖父层View布局，各层View实现相对布局后，每当屏幕尺寸改变时各层View会自动适配，同时容器的组件列数和尺寸会按响应式规则进行适配，一套代码即可适配所有屏幕尺寸，实现响应式布局。

### 三 优酷响应式成果

目前优酷全端已具备响应式布局的能力，八月份已上线universal版本，一套代码支持iPhone、iPad竖屏、iPad横屏、浮窗、各种比例分屏，为用户提供了更好更丰富的用户体验。

![image.png](https://segmentfault.com/img/remote/1460000037523323)

![image.png](https://segmentfault.com/img/remote/1460000037523324)

### 四 总结

响应式能力是多端投放能力的第一步，优酷实现响应式布局后对开发、设计和产品都提出了更高的要求，同时鉴于iPad低端设备占比较高，业务开发过程中不仅要考虑通投能力，更要求App始终保持更高的性能和稳定性，这是我们持续在努力的。

苹果2020年底将推出基于ARM架构的MacBook，也有媒体曝光，苹果正在申请折叠屏相关的专利，相信未来苹果设备的尺寸会越来越丰富，App适配提效是绕不开的话题，而优酷响应式的开发极大扩展了iPhone版App的适用场景，是解决多种设备支持的更好途径，为适应未来更复杂的设备场景打下坚实基础。
