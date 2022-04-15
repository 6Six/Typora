# [阿里自研标准化协议库 XQUIC 正式开源](https://mp.weixin.qq.com/s/OrvjikvYyqlHAzGamLmpsw)



XQUIC[1]是阿里自研的IETF QUIC标准化传输协议库。**XQUIC是基于IETF QUIC协议实现的UDP传输框架**，包含加密可靠传输、HTTP/3两大块主要内容，为应用提供可靠、安全、高效的数据传输功能，可以极大改善弱网和移动网络下产品的用户网络体验。这项技术研发由大淘宝平台技术团队发起和主导，当前有达摩院XG实验室、阿里云CDN等多个团队参与其中。



现今QUIC有多家开源实现，为什么选择标准协议 + 自研实现的道路？我们从14年开始关注Google在QUIC上的实践（手机淘宝在16年全面应用HTTP/2），从17年底开始跟进并尝试在电商场景落地GQUIC[2]，在18年底在手淘图片、短视频等场景落地GQUIC并拿到了一定的网络体验收益。然而在使用开源方案的过程中，或多或少碰到了一些问题，例如包大小过大、依赖复杂等等。最终促使我们走上自研实现的道路。



为什么要选择IETF QUIC[3]标准化草案的协议版本？过去我们也尝试过自研私有协议，在端到端都由内部控制的场景下，私有协议的确是很方便的，并且能够跟随业务场景的需求快速迭代演进；但私有协议方案很难走出去建立一个生态圈 / 或者与其他的应用生态圈结合（遵循相同的标准化协议实现互联互通）；另一方面从云厂商的角度，私有协议也很难与外部客户打通；同时由于IETF开放讨论的工作模式，协议在安全性、扩展性上会有更全面充分的考量。因此我们选择IETF QUIC标准化草案版本来落地。截止目前，IETF工作组已经发布QUIC v1版本[4]RFC，XQUIC已经支持该版本，并能够与其他开源实现基于QUIC v1互通。



**XQUIC 的优势**



![图片](https://mmbiz.qpic.cn/mmbiz_png/KEg9e0yplfr7OTSyczV91ibXd5lZPGnIMR3jfpMKlI4tGnicxCBj3IxMZFrvQvck0YoYLzNhOiaNwNkdQWwkLTueg/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)



XQUIC是一个轻量、高性能、标准化的跨平台协议库：



轻量性：

- XQUIC在Android/iOS双端的编译产物均小于400KB
- 除TLS/1.3能力依赖SSL库之外，无其他外部依赖，可以方便地部署到移动设备和各种嵌入式设备中

- 适用于需要高性能但同时又对包大小敏感的移动端APP场景（为了减少新用户的安装成本，移动端APP希望能尽量减少APP包大小）



高性能传输：

- XQUIC已经在手机淘宝实现核心导购、短视频链路大规模使用，并相对于内核态TCP+HTTP/2优化20%的网络请求耗时
- 支持0-RTT功能

- 支持多通道传输加速能力[5]



标准化：

- XQUIC实现了整套IETF QUIC标准协议，包含传输层、加密层、应用层协议栈
- 协议版本支持QUIC version 1，以及draft-29

- SSL库兼容适配BoringSSL或BabaSSL（可任意选择其中之一）



易用性：

- 跨平台：支持Linux/Android/iOS/Mac等平台，后续也会支持Windows平台适配，客户端可以通过SDK方式很方便地接入并使用。
- 支持Wireshark解析、qlog事件日志标准，方便问题排查

- 完善的文档（中文/英文对照）、demo示例和单测



**XQUIC 核心介绍**

**模块设计**





XQUIC是IETF QUIC草案版本的一个C协议库实现，端到端的整体链路架构设计如下图所示。XQUIC内部包含了QUIC-Transport（传输层）、QUIC-TLS（加密层、与TLS/1.3对接）和HTTP/3.0（应用层）的实现。除了每层的协议栈功能模块之外，在公共模块部分，XQUIC也支持了qlog[5]日志标准。



![图片](https://mmbiz.qpic.cn/mmbiz_png/KEg9e0yplfr7OTSyczV91ibXd5lZPGnIM7rGdSV31MyRNGa9mlb7qcHy2knibqFPfnsPzbDVbWBRV6Bt8o4ekQAw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)



**拥塞控制算法框架**





![图片](https://mmbiz.qpic.cn/mmbiz_png/KEg9e0yplfr7OTSyczV91ibXd5lZPGnIMibQomprIwUl0aQU8zuOlaOqTFmVGgJdzEbedbNwWkfVicoOgSCEzsOKA/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)



拥塞控制算法模块，在传输协议栈中承担了发动机的职能。为了能够方便地实现多套拥塞控制算法、并方便针对各类典型场景进行优化，我们将拥塞控制算法流程抽象成7个回调接口，其中最核心的两个接口onAck和onLost用于让算法实现收到报文ack和检测到丢包时的处理逻辑。XQUIC内部实现了多套拥塞控制算法，包括最常见的Cubic、New Reno，以及时下比较流行的BBR v1和v2，每种算法都只需要实现这7个回调接口即可实现完整算法逻辑。



为了方便用数据驱动网络体验优化，我们将连接的丢包率、RTT、带宽等信息通过采样和分析的方式，结合每个版本的算法调整进行效果分析。同时在实验环境下模拟真实用户的网络环境分布，更好地预先评估算法调整对于网络体验的改进效果。



**传输层能力和应用协议协商**





XQUIC提供两套接口，分别是使用标准HTTP3的7层接口和直接使用传输层能力的4层接口，同时XQUIC支持ALPN[6]协商机制，可以通过向ALPN接口注册新的应用层协议回调，并通过握手期间的协商实现多套应用层协议的兼容。



![图片](https://mmbiz.qpic.cn/mmbiz_png/KEg9e0yplfr7OTSyczV91ibXd5lZPGnIMWUQSVpa4HVvuNGkjibonUKgpAHDoBIia8Qy7R3GnMtibYI91kZmdsmSZA/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)



**7层协议的扩展能力和易用性：**XQUIC的接口中，将QUIC Transport事件归类为通用传输层事件和面向应用层协议的事件。连接会话、Stream事件面向Application-Layer-Protocol定义；而剩余的通用传输层事件，因为在不同的应用层协议之间，具备高度共性，可以复用。这种设计保证了在扩展多种7层协议的时候，开发者只需要关注7层协议对于连接会话、Stream数据的处理，而不需要重复对QUIC传输层通用事件进行开发。



**TLS层设计**





![图片](https://mmbiz.qpic.cn/mmbiz_png/KEg9e0yplfr7OTSyczV91ibXd5lZPGnIM6okEZuDQ6DfzBEA0IxQgXS6bQDyqiaDYYjeiabOTicjsxIYrl4kcYU9kg/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)



QUIC Transport层，对TLS模块有如下依赖：加密握手协商、数据加解密、密钥更新、session resumption、0-RTT、传输参数、ALPN协商。TLS层，则需要依赖底层SSL库，来支撑上述功能。因此，TLS模块存在数据的多样性，以及依赖的多样性，数据流程和代码结构会比较复杂。TLS层需要对这些数据流进行归类整理，从而来简化上下游的依赖关系，降低代码的复杂度。



XQUIC适配了babassl、boringssl两种底层的ssl库，向上提供统一的接口，从而消除了它们之间接口、流程的差异，并抽象为统一的内部数据流程，仅针对不同ssl库提供轻薄的适配层，减少重复适配的代码逻辑，达到降低代码复杂度、提升可维护性的效果。同时XQUIC也提供了编译选项，方便开发者根据自身应用的情况，选择适合自己的依赖库。



**XQUIC 开源历史**

**为什么要做 XQUIC**





![图片](https://mmbiz.qpic.cn/mmbiz_png/KEg9e0yplfr7OTSyczV91ibXd5lZPGnIMOwU0gFFrstriboMrw8GcyxwibibLw3YpKO2mUNHeNDC95rZGkVIBRicSicg/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)



我们从18年左右，开始探索从TCP转向UDP方向，最早是基于GQUIC，主要应用在手淘的图片和短视频等内容分发的场景。在18年底19年初，当时大家有一个共同的判断是要走标准化道路，一方面整个标准化协议的设计和安全性都有更完备的考量，另一方面是因为从网络加速产品角度，私有协议解决方案更难被用户认可。在决定选择标准化道路之后，当时市面上也没有特别成熟并适用于移动端的IETF QUIC协议栈实现，所以手淘就启动了自研XQUIC项目。



经过1年半的研发和打磨，于20年的6月份开始全面上线，并在20年8月在手淘核心导购RPC请求场景进行规模化验证。在21年初与CDN IETF QUIC产品实现对接，并在短视频场景上开始逐步应用IETF QUIC技术。在去年的9月份我们实现了IETF QUIC整套协议栈在短视频场景下的规模化应用。之后，我们经历了2021年双十一的考验，XQUIC的性能和稳定性都有了很好的验证，因此在今年的1月7号，我们完成XQUIC的对外开源，后续也将持续更新迭代开源版本。



**我们为什么要开源 XQUIC**





通过开源可以帮助整个社区更好地了解这项技术，可以帮助我们改进，同时可以通过社区的影响力对这项技术加以推广。社区的反馈也能够帮助我们吸收更多的需求场景输入，帮助我们更好地迭代这项技术。我们期望XQUIC在服务于淘宝技术的同时积极回馈社会，也欢迎网络技术研发的爱好者加入开源社区与我们交流。



**应用场景和效果**



![图片](https://mmbiz.qpic.cn/mmbiz_png/KEg9e0yplfr7OTSyczV91ibXd5lZPGnIMQnZbS5JqJgcR2DvriaaIPcoxibiaTnbvuUo5Mm7oMDBIBzkdNa6zJz7bw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)



目前，XQUIC已经在手淘Android/iOS双端正式版本、以及集团统一接入网关大规模应用，比如我们打开手机淘宝的首页，或是搜索我们感兴趣的商品，或是打开逛逛浏览达人的视频，XQUIC都为这些场景提供更快的网络数据传输，每天稳定为超过百亿量级的网络请求提供端到端加速能力。在2021年的双十一购物节中，XQUIC在核心导购链路、短视频场景下也经过了大规模验证。



![图片](https://mmbiz.qpic.cn/mmbiz_png/KEg9e0yplfr7OTSyczV91ibXd5lZPGnIMl0d1fp4GUy3JPGymfUh8NEgR972rhJNoXA7BCt7tpdK5pR4D9JX8YQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)



**后续 Roadmap**



我们计划每1~2个月发布一个稳定版本，当前计划如下：



**新功能特性方面：**

- 互通性功能补充，包括Key update、Retry以及ECN

- WG draft版本的多路径功能支持

- 适配开源Tengine的module支持

- 非可靠传输datagram支持

- Masque特性支持

  



由于当前Multi-path QUIC[5]草案正处于即将被IETF QUIC Working Group接收的流程中，并且WG draft版本与XQUIC前期支持的多路径版本有部分差异，因此我们暂时在开源版本去掉了这部分功能。后续我们将在2月份基于Working Group草案版本更新多路径功能。



**性能优化方面：**

- UDP性能优化feature
- XUDP适配支持



**跨平台支撑方面：**

- windows平台支持



**配套工具方面：**

- 网络性能测量工具



**文档及中文资料：**

- 开源仓库已经提供了基于draft-34校订的草案中文翻译，后续会陆续更新RFC8999-9002的中文译版



**附录：**

[1] XQUIC: https://github.com/alibaba/xquic

[2] GQUIC: 指Google QUIC版本，与IETF QUIC草案版本有一定差异

[3] IETF QUIC: 指IETF工作组推进的QUIC标准 https://datatracker.ietf.org/wg/quic/documents/，包括RFC8999~9002，以及还在推进中的HTTP/3.0、QPACK等一系列内容

[4] QUIC v1: RFC8999~9002所描述的QUIC协议版本

[5] MPQUIC: 即将被IETF QUIC WG工作组接收的草案 https://datatracker.ietf.org/doc/draft-lmbdhk-quic-multipath/

[6] ALPN协商: https://datatracker.ietf.org/doc/html/rfc7301