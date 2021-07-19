# 架构篇

## 前言

上一篇[移动应用遗留系统重构（1）- 开篇](https://juejin.cn/post/6943470229905211422)我们分享了移动应用遗留系统常见的问题。那么好的实践或者架构设计是怎样的呢？

这一篇我们将整理业内优秀的移动应用架构设计，包含微信、淘宝、支付宝以及美团外卖。其中的部分产品也经历过遗留系统的重构改造，具有非常好的参考意义。

## 优秀实践

### 微信

从微信对外分享的架构演进文章中可知，微信应用其实也是经历了从大单体到模块化的演进。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/268ddddfad6e430db81c0c91d7982116~tplv-k3u1fbpfcp-zoom-1.image)

> 图片来源 [微信Android模块化架构重构实践](https://mp.weixin.qq.com/s/6Q818XA5FaHd7jJMFBG60w)

我们看下介绍中后续改造后的架构设计。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/89aa2b911e0c424da5daeb1723b1a465~tplv-k3u1fbpfcp-zoom-1.image)

> 图片来源 [微信Android模块化架构重构实践](https://mp.weixin.qq.com/s/6Q818XA5FaHd7jJMFBG60w)

设计中提到重构主要3个目标

* 改变通信方式 （API化）
* 重新设计模块 （中心化业务代码回归各自业务）
* 约束代码边界 （pins工程结构，更细粒度管控边界）

我们可以发现重构后架构比原来的单体应用的一些变化。

1. 业务模块独立编译调试，耦合度低
2. 代码复用高，有统一公共的组件库及kernel
3. 模块职责、代码边界清晰，强约束

更多信息可阅读原文，[微信Android模块化架构重构实践](https://mp.weixin.qq.com/s/6Q818XA5FaHd7jJMFBG60w)

### 手淘

从手机淘宝客户端架构探索实践的分享中介绍到手机淘宝从1.0用单工程编写开始，东西非常简陋；到2.0为索引许多三方库的庞大的单工程；再到3.0打破了单工程开发模式实现业务复用。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/1d4a21eaed824140924a41b49f310b04~tplv-k3u1fbpfcp-zoom-1.image)

> 图片来源 [手机淘宝客户端架构探索实践](https://yq.aliyun.com/articles/129)

淘宝架构主要分为四层，最上层是组件Bundle\(业务组件\)，往下是容器\(核心层\)，中间件Bundle\(功能封装\)，基础库Bundle\(底层库\)。

文章提到架构演化的一些优点及变化很值得深思。

1. 业务复用，减少人力
2. 基础复用，做深做精
3. 敏捷开发，快速试错

### 支付宝

在支付宝mPass实践讨论分析一文中，提到支付宝客户端的总体架构图如下。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/0b356a0929a04d22bc3fc0a9fc868303~tplv-k3u1fbpfcp-zoom-1.image)

> 图片来源 [开篇 \| 模块化与解耦式开发在蚂蚁金服 mPaaS 深度实践探讨](https://mp.weixin.qq.com/s?__biz=MzUyMDk2MzUzMQ==&mid=2247483672&idx=1&sn=249447e402bd41e455f3990d210d9c03&scene=21#wechat_redirect)

分享文章中介绍到5层架构设计如下：

* 最底层是支付宝框架的容器层，包括类加载资源加载和安全模块；
* 第二层是我们抽离出来的组件层，包括网络库，日志库，缓存库，多媒体库，日志等等，简单说这些是一些通用的能力层；
* 第三层是我们定制的框架层，这是关键部分，是我们得以实现上千人，上千多个工程共同开发一个 App 的基础。
* 第四层是基于框架封装出来的业务服务层；
* 第五层便是具体的业务模块，其中每一个模块都是一个或多个具体的工程;

文章中介绍到关于工程之间的依赖关系的处理比较特别。

在支付宝的架构里，编译参与的部分是和运行期参与的部分是分离的：编译期使用 bundle 的接口包，运行期使用 bundle 包本身。bundle 的接口包是 bundle 包的一部分，即刚才说的 bundle 的代码部分。bundle 的资源包同时打进接口包，在编译期提供给另一个 bundle 引用。

更多信息可阅读原文，[开篇 \| 模块化与解耦式开发在蚂蚁金服 mPaaS 深度实践探讨](https://mp.weixin.qq.com/s?__biz=MzUyMDk2MzUzMQ==&mid=2247483672&idx=1&sn=249447e402bd41e455f3990d210d9c03&scene=21#wechat_redirect)

### 美团

最后看另外一个跨平台技术架构相关的分享，在外卖客户端容器化架构的演进分享中提到了美团外包的整体架构如下。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/9bbefe79b121441eaccd2d13b038a10c~tplv-k3u1fbpfcp-zoom-1.image)

> 图片来源 [外卖客户端容器化架构的演进](https://tech.meituan.com/2020/09/30/waimai-mobile-architecture-evolution.html)

特别的一点是是采用了容器化架构，根据业务场景及PV，支持多种容器技术。在文末的总结提到，容器化架构相对于传统的移动端架构而言，充分地利用了现在的跨端技术，将动态化的能力最大化的赋予业务。通过动态化，带来业务迭代周期缩短、编译的加速、开发效率的提升等好处。同时，也解决了面临着的多端复用、平台能力、平台支撑、单页面多业务团队、业务动态诉求强等业务问题。但对线上的可用性、容器的可用性、支撑业务的线上发布上提出了更加严格的要求。

更多信息可阅读原文，[外卖客户端容器化架构的演进](https://tech.meituan.com/2020/09/30/waimai-mobile-architecture-evolution.html)

## 总结

**架构是为了解决业务的问题，没有银弹。** 但通过这些业内的优秀实践分享，我们可以发现一些优秀的设计范式。

1. 代码复用
2. 公共能力复用，有专门统一管理应用公用的基础能力，如图片、网络、存储能力、安全等
3. 公用业务能力复用，有专门统一管理应用的业务通用组件，如分享、推送、登录等
4. 低耦合，高内聚
5. 业务模块间通过API方式依赖，不依赖具体的模块实现
6. 依赖方向清晰，上层模块依赖下层模块
7. 并行研发
8. 业务模块支持独立编译调试
9. 业务模块独立发布

结合这些特点及CloudDisk团队的业务，团队采用的架构设计如下。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/87df48f3c43149f7b54b4a23d58a7d78~tplv-k3u1fbpfcp-zoom-1.image)

下一篇，移动应用遗留系统重构（3）- 示例篇，我们将继续介绍CloudDisk的业务及团队问题，分析现有的代码。

## 参考

[微信Android模块化架构重构实践](https://mp.weixin.qq.com/s/6Q818XA5FaHd7jJMFBG60w)

[手机淘宝客户端架构探索实践](https://yq.aliyun.com/articles/129)

> 参考来自阿里云开发者社区，但链接已失效

[开篇 \| 模块化与解耦式开发在蚂蚁金服 mPaaS 深度实践探讨](https://mp.weixin.qq.com/s?__biz=MzUyMDk2MzUzMQ==&mid=2247483672&idx=1&sn=249447e402bd41e455f3990d210d9c03&scene=21#wechat_redirect)

[外卖客户端容器化架构的演进](https://tech.meituan.com/2020/09/30/waimai-mobile-architecture-evolution.html)

