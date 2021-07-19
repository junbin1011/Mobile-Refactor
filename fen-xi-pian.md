# 分析篇

## 前言

上一篇[移动应用遗留系统重构（3）-示例篇](https://juejin.cn/post/6947855094272491556)我们介绍了CloudDisk的业务及代码现状。分享了“理想”（未来的架构设计）与“现实”（目前的代码现状），接下来在我们开始动手进行重构时，我们首先得知道往理想的设计架构演化，中间存在多少问题。一方面作为开始重构的输入，另外一方面我们有数据指标，也能更好评估工作量及衡量进度。

接下来我们将根据架构篇团队采用的架构设计，结合目前的代码，总结分析工具及方法。

## 架构设计

我们先回忆一下[架构篇](https://juejin.cn/post/6945313969556946980)里团队采用的架构设计。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ef9a448908e64747bd7d43df1777a006~tplv-k3u1fbpfcp-zoom-1.image)

1. 代码复用
2. 公共能力复用，有一层专门统一管理应用公用的基础能力，如图片、网络、存储能力、安全等
3. 公用业务能力复用，有一层专门统一管理应用的业务通用组件，如分享、推送、登录等
4. 低耦合
5. 业务模块间通过API方式依赖，不依赖具体的模块实现
6. 依赖方向清晰，上层模块依赖下层模块
7. 并行研发
8. 业务模块支持独立编译调试
9. 业务模块独立发布

结合该4层架构、已有的代码，以及业务的后续演化，团队设计的新架构如下

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ee03da85960a4f719d42bbfe7faa24d5~tplv-k3u1fbpfcp-zoom-1.image)

## 分析工具

### [ArchUnit](https://www.archunit.org/)

有了架构设计后，我们就能识别代码的边界，这里我们可以通过[Archunit](https://www.archunit.org/)进行边界约束描述。我们可以得到2条通用的守护规则。

1. 垂直方向，下层模块不能反向依赖上层
2. 横向方向，组件之间不能存在相互的依赖

转化为ArchUnit的测试用例如下:

```java
  @ArchTest
    public static final ArchRule architecture_layer_should_has_right_dependency =layeredArchitecture()
            .layer("Library").definedBy("..cloud.disk.library..")
            .layer("PlatForm").definedBy("..cloud.disk.platform..")
            .layer("FileBundle").definedBy("..cloud.disk.bundle.file..")
            .layer("DynamicBundle").definedBy("..cloud.disk.bundle.dynamic..")
            .layer("UserBundle").definedBy("..cloud.disk.bundle.user..")
            .layer("AllBundle").definedBy("..cloud.disk.bundle..")
            .layer("App").definedBy("..cloud.disk.app..")
            .whereLayer("App").mayOnlyBeAccessedByLayers()
            .whereLayer("FileBundle").mayOnlyBeAccessedByLayers("App")
            .whereLayer("DynamicBundle").mayOnlyBeAccessedByLayers("App")
            .whereLayer("UserBundle").mayOnlyBeAccessedByLayers("App")
            .whereLayer("PlatForm").mayOnlyBeAccessedByLayers("App","AllBundle")
            .whereLayer("Library").mayOnlyBeAccessedByLayers("App","AllBundle","PlatForm");
```

当然这个用例的执行是失败的，因为我们基本的包结构还没有调整。但有了架构守护测试用例，我们就可以逐步把代码移动到对应的Package中，直到守护用例运行通过为止。

接下来我们先运用IDE工具进行基础的包结构调整，调整后的结构如下

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/1e62dd4db6a14118a4900a3ac34d774f~tplv-k3u1fbpfcp-zoom-1.image)

调整后运行ArchUnit测试运行结果如下 ![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/1706451a359646c19b2a4ef4cb0e1daf~tplv-k3u1fbpfcp-zoom-1.image)

这些异常的提示就是我们需要处理的异常依赖。但是ArchUnit的这个提示比较不不友好，接下来我们介绍另外一种分析异常依赖的方式，使用Intellij Dependencies 。

### Intellij Dependencies

我们选择对应的Package，选择Analyze菜单，点击Dependencies，可以找出该Package所依赖的相关类。 ![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/5b56ef5706064b22b44a559b4bd71ac5~tplv-k3u1fbpfcp-zoom-1.image)

我们选择dynamic Package进行分析后，发现根据现有的架构约束，存在横向的Bundle依赖需要进行解除依赖。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e0c6ff5913084b3cba7061bdbdbe56c1~tplv-k3u1fbpfcp-zoom-1.image)

我是在实际重构过程中，我们可以频繁借助该功能验证耦合解除情况，并且同时通过ArchUnit测试做好守护。

**详细代码见**[**Cloud Disk**](https://github.com/junbin1011/CloudDisk/commit/d4b81762177b3a5e38d9d622170786540bcc7478)

## 总结

这一篇我们分享了如何借助工具进行异常依赖的分析。当我们有了未来的架构设计后，可以借助ArchUnit进行架构测试守护，通过Intellij的Dependendencies 我们可以方便以Package或者Class为单位进行依赖分析。

当我们已经分析出需要处理的异常依赖，接下来我们就可以逐步进行重构。下一篇，我们将给大家分享实践总结的一些重构套路，移动应用遗留系统重构（5）- 重构方法篇。

