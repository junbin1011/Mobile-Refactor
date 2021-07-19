# 依赖注入篇

## 前言

上一篇[移动应用遗留系统重构（7）- 解耦重构演示篇\(一\)](https://juejin.cn/post/6959504791642832909)我们对file包进行了重构，抽取了对应的UserState接口，但我们发现UserState接口的层层传递，我们需要手工维护好多的构造方法及对应的注入，这样非常不便于进行代码的管理及维护。同时随着解耦的接口越来越多，就会产生更多的样板代码，所以我们需要更好的方式进行统一的管理。

这篇我们主要分为3个部分，第一部分是常见依赖注入方式，第二部分是业内优秀的依赖注入实践，最后我们将继续对CloudDisk进行依赖注入的重构。

## 依赖注入方式

### 静态注入（在编译时连接依赖项的代码）

静态注入是最常用的方式，在类中依赖抽象，暴露接缝，在调用的地方进行实现的注入。常用的注入方式有2种。

1. 构造函数注入。您将某个类的依赖项传入其构造函数（上一篇的CloudDisk就是采用这种方式）。
2. 字段注入（或 setter 注入）。某些 Android 框架类（如 Activity 和 Fragment）由系统实例化，因此无法进行构造函数注入。使用字段注入时，依赖项将在创建类后实例化

由于是编译时连接依赖项，所以编辑阶段会进行类型的检查。

### 动态注入 （在运行时连接依赖项）

动态注入最常见的方式就是通过反射的机制，在运行时进行注入。

由于是运行时连接依赖项，所以编译阶段没有检查，且有反射带来的性能问题。但好处是灵活，如果有模块是动态加载的，利用这种方式处理起来更简单。

### 对比

| 静态注入 | 动态注入 |
| :--- | :--- |
| 类型安全，编译时检查 | 运行时绑定，编译时没有依赖 |
| 性能无损失 | 有反射带来的性能损失 |
| 较适合整包编译 | 较适合模块动态加载场景 |
| 开源实现多 | 开源实现较少 |

## 业内优秀实践

前面提到适用手工的方式进行依赖注入的管理，是一项非常困难和有挑战的事情。所幸，业内已经有成熟的解决方案。这一章我们来看下业内优秀的实践。

### [hilt](https://developer.android.com/training/dependency-injection)

Hilt 采用的是静态注入的方式，在依赖项注入库[Dagger](https://dagger.dev/) 的基础上构建而成，提供了一种将 Dagger 纳入 Android 应用的标准方法。

详细的介绍及使用大家可以查看官方的介绍说明

[https://developer.android.com/training/dependency-injection/hilt-android](https://developer.android.com/training/dependency-injection/hilt-android)

这里我们重点分享几点这个库的一些优点。

1. 相比Dragger，使用其实更简单
2. IDE支持链接跳转，开发体验好
3. 完整的测试套件，方便进行自动化测试编写
4. Jetpack生态组件，生态链完整，社区活跃度搞

### [koin](https://insert-koin.io/)

Koin 是一个用于 Kotlin 的实用型轻量级依赖注入框架，采用纯 Kotlin 编写而成，仅使用功能解析，无代理、无代码生成、无反射。

详细的介绍及使用同样大家可以查看官方的介绍说明

[https://insert-koin.io/docs/quickstart/kotlin](https://insert-koin.io/docs/quickstart/kotlin)

这里我们同样分享几点这个库的一些优点。

1. 使用简单，没有复杂的注解
2. 更轻量，相比hilt编译时间更快、生成代码更少

当然，由于koin本身是用kotlin语言编写的，所以最好项目也是使用kotlin编写。相比hilt，没有那么细的生命周期管理以及IDE的支持，并且也没有Jetpack的生态丰富。

## CloudDisk依赖注入重构示例

前面我们分享了业内一些优秀的依赖注入实践，由于CloudDisk是采用java语言开发，且考虑后续会往JetPack生态迁移，所以决定采用Hilt的方案。

具体的Hilt改造过程我们就不演示，[参考官网的文档](https://developer.android.com/training/dependency-injection/hilt-android)进行操作则可。这里我们对比一下改造前后的代码片段。

具体的代码：[github链接](https://github.com/junbin1011/CloudDisk/commit/5a0cbec49216c7ab43e88a2ebbed51f549f90cfd)

使用手工注入：

```text
public class FileFragment extends Fragment {

    FileController fileController;

    public FileFragment(UserState userState) {
        fileController = new FileController(userState);
    }

    public static FileFragment newInstance(UserState userState) {
        FileFragment fragment = new FileFragment(userState);
        Bundle args = new Bundle();
        fragment.setArguments(args);
        return fragment;
    }
}
```

使用Hilt注入：

```text
@AndroidEntryPoint
public class FileFragment extends Fragment {

    @Inject
    FileController fileController;

    public static FileFragment newInstance() {
        FileFragment fragment = new FileFragment();
        Bundle args = new Bundle();
        fragment.setArguments(args);
        return fragment;
    }
```

> Hilt本质上通过注解及gradle插件，在编译时生成代码及注入到我们的类中。使用框架帮我们减少了很多模板代码，统一配置管理。

这里注意，测试代码也需要做相应的配置。

```text
@RunWith(AndroidJUnit4.class)
@LargeTest
@HiltAndroidTest
@Config(application = HiltTestApplication.class)
public class SmokeTesting {
    @Rule
    public HiltAndroidRule hiltRule = new HiltAndroidRule(this);
}
```

## 总结

本章我们介绍了常见的依赖注入方式及业内优秀的实践，同时我们也将CloudDisk进行了改造，使用Hilt统一管理注入。通过IDE的依赖分析我们可以发现，App依赖了fileBundle的Fragment，UI上存在编译的依赖。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e648106d39f0429995249f8681838648~tplv-k3u1fbpfcp-zoom-1.image)

下一篇，移动应用遗留系统重构（9）- 路由篇，我们将分享常见的页面路由方式及业内优秀的实践，并对DiskCloud继续进行改造优化。

## 参考资料

[Android 中的依赖项注入](https://developer.android.com/training/dependency-injection)

