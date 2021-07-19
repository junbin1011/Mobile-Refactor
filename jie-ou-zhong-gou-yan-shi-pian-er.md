# 解耦重构演示篇（二）

## 前言

在[移动应用遗留系统重构（8）- 依赖注入篇 ](https://juejin.cn/post/6963214120178941983)、[移动应用遗留系统重构（9）- 路由篇](https://juejin.cn/post/6966166367821103117)章节中，我们已经完成了基础的注入和路由框架搭建。接着[移动应用遗留系统重构（7）- 解耦重构演示篇\(一\)+视频演示 ](https://juejin.cn/post/6959504791642832909),本篇我们会把App中剩余的 platform 包、dynamic 包进行重构。文中会主要列出分析及解耦的思路及过程,并且会有详细的完整演示视频。

**platform包重构代码演示：** [https://mp.weixin.qq.com/s/YJLBFBD9T6F0zW7hWINoZA](https://mp.weixin.qq.com/s/YJLBFBD9T6F0zW7hWINoZA)

**dynamic 包重构代码演示：** [https://mp.weixin.qq.com/s/ZcDDIrJwUbz5JCgrzjfteg](https://mp.weixin.qq.com/s/ZcDDIrJwUbz5JCgrzjfteg)

## 安全重构演示

### platform 包重构

1. 依赖分析

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/835894a6cc314ec4bf454753bf1efd60~tplv-k3u1fbpfcp-zoom-1.image)

platform 中存在对上层bundle中的反向依赖，该依赖需要解除后，便可将该包下沉到platform的模块中。

1. 安全重构

重构前代码：

```text
 public class LoginActivity extends AppCompatActivity {

    UserController userController = new UserController();

    @Override
    public void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_login);
        userController.login("", "", new CallBack() {
            @Override
            public void success(String message) {

            }

            @Override
            public void filed(String message) {

            }
        });
    }
}
```

重构手法：**提取代理类、内联、移动**

* 由于UserControler中存在login及getUserInfo方法，我们不能简单将类一起下沉，需要抽取独立类将login方法和LoginActivity一起内聚下沉

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/4faf078597ec43bf94099b56f49b6175~tplv-k3u1fbpfcp-zoom-1.image)

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/6083094a1b624a978f4d05916b624f5a~tplv-k3u1fbpfcp-zoom-1.image)

重构后代码：

```text
public class UserController {
    public static boolean isLogin = false;
    public final LoginController loginController = new LoginController();

    public boolean login(String id, String password, CallBack callBack) {
        //用户登录
        return loginController.login(id, password, callBack);
    }

    public UserInfo getUserInfo() {
        //获取用户信息
        return loginController.getUserInfo();
    }
}
```

* 将LoginActivity对UserController的调用内联为对LoginController的调用

内联login方法： ![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/1dadcca3d5e84162a7c638841819cf53~tplv-k3u1fbpfcp-zoom-1.image)

内联loginController： ![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/5d3b650158144c37bc44b976612d9bb1~tplv-k3u1fbpfcp-zoom-1.image)

抽取成员变量：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/06715830741c49959f3aa30ab2e16834~tplv-k3u1fbpfcp-zoom-1.image)

重构后代码：

```text
public class LoginActivity extends AppCompatActivity {

    private LoginController loginController = new LoginController();

    @Override
    public void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_login);
        //用户登录
        loginController.login("", "", new CallBack() {
            @Override
            public void success(String message) {

            }

            @Override
            public void filed(String message) {

            }
        });
    }
}
```

1. 代码移动

代码移动至独立的platform，加上对应的Gradle依赖：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/cf1b0b2a2cba466590d894187cba2cc9~tplv-k3u1fbpfcp-zoom-1.image)

1. 功能验证

   执行冒烟测试，验证功能

```text
./gradlew app:testDeb
ug --tests SmokeTesting
```

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a4d480b8756a46b3ad022e7a6ce9a42e~tplv-k3u1fbpfcp-zoom-1.image) **代码演示：** [https://mp.weixin.qq.com/s/YJLBFBD9T6F0zW7hWINoZA](https://mp.weixin.qq.com/s/YJLBFBD9T6F0zW7hWINoZA)

具体的代码：[github链接](https://github.com/junbin1011/CloudDisk/commit/0adcc7d7b8fac54ede058420b5be56fc9e0629dc)

### dynamic 包重构

1. 依赖分析

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/5d0abc6f42674f799ac97d24393b3ae9~tplv-k3u1fbpfcp-zoom-1.image)

dynamic包存在对fileBundle和userBundle之间的横向依赖。主要是因为动态中需要上传和下载文件，所以依赖了fileBundle，另外需要判断是否登录，所以也依赖了userBundle

1. 安全重构

重构前代码：

```text
public class DynamicFragment extends Fragment {

    DynamicController dynamicController = new DynamicController();
    FileController fileController = new FileController(new UserStateImpl());
    Button btnShare;
    public static DynamicFragment newInstance() {
        DynamicFragment fragment = new DynamicFragment();
        Bundle args = new Bundle();
        fragment.setArguments(args);
        return fragment;
    }

    @Override
    public void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        dynamicController.getDynamicList();
        FileInfo fileInfo = fileController.upload("/data/data/user.png");
        dynamicController.post(new Dynamic(), fileInfo);
    }
}

public class DynamicController {

    FileController fileController=new FileController(new UserStateImpl());

    public boolean post(Dynamic dynamic, FileInfo fileInfo) {
        //发送一条动态消息
        if (!UserController.isLogin) {
            return false;
        }
        HttpUtils.post("http://dynamic", LoginController.userId);
        return true;
    }

    public List<Dynamic> getDynamicList() {
        //通过网络获取动态信息,有些动态带有附件需要下载
        fileController.download("");
        return new ArrayList<>();
    }
}
```

重构手法：**提前代理类、抽取接口、移动、内联、提取变量等**

由于FileController中除了上传和下载还有获取文件的方案，我们不能简单粗暴就把整个FileController提前接口，希望抽取除了的接口职责更加单一。

* 抽取FileTransfer类，提取接口
* 将对FileController的依赖内联为FileTransfer
* 对UserBundle的依赖使用已经抽取的UserState接口
* 使用注入的方式进行依赖管理

> 由于步骤比较多，大家可以直接看视频的演示，这里就不一一截图说明。

重构后代码：

```text
@AndroidEntryPoint
public class DynamicFragment extends Fragment {

    @Inject
    DynamicController dynamicController;
    Button btnShare;
    @Inject
    TransferFile transferFile;

    public static DynamicFragment newInstance() {
        DynamicFragment fragment = new DynamicFragment();
        Bundle args = new Bundle();
        fragment.setArguments(args);
        return fragment;
    }

    @Override
    public void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        dynamicController.getDynamicList();
        //上传文件
        FileInfo fileInfo = transferFile.upload("/data/data/user.png");
        dynamicController.post(new Dynamic(), fileInfo);
    }
}

public class DynamicController {

    @Inject
    TransferFile transferFile;
    @Inject
    UserState userState;

    @Inject
    public DynamicController() {
    }

    public boolean post(Dynamic dynamic, FileInfo fileInfo) {
        //发送一条动态消息
        if (!userState.isLogin()) {
            return false;
        }
        HttpUtils.post("http://dynamic", LoginController.userId);
        return true;
    }

    public List<Dynamic> getDynamicList() {
        //通过网络获取动态信息,有些动态带有附件需要下载
        //下载文件
        transferFile.download("");
        return new ArrayList<>();
    }
}
```

1. 代码移动

代码移动至独立的dynamic Bundle，加上对应的Gradle依赖：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/274fba7c32df436b97d0130c613e6b56~tplv-k3u1fbpfcp-zoom-1.image)

1. 功能验证

   执行冒烟测试，验证功能

```text
./gradlew app:testDeb
ug --tests SmokeTesting
```

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/4b6573ea58224f6f892f6f8f7625e794~tplv-k3u1fbpfcp-zoom-1.image)

**代码演示：**[https://mp.weixin.qq.com/s/ZcDDIrJwUbz5JCgrzjfteg](https://mp.weixin.qq.com/s/ZcDDIrJwUbz5JCgrzjfteg)

具体的代码：[github链接](https://github.com/junbin1011/CloudDisk/commit/a1067bfca48e2f2e5a59e1bda94a2c56e7b55f5c)

## ArchUnit

在重构的过程中，我们补充了api的模块，我们需要调整ArchUnit的用例。

```text
@RunWith(ArchUnitRunner.class)
@AnalyzeClasses(packages = "com.cloud.disk")
public class ArchRuleTest {

    @ArchTest
    public static final ArchRule architecture_layer_should_has_right_dependency =layeredArchitecture()
            .layer("Library").definedBy("..cloud.disk.library..")
            .layer("Api").definedBy("..cloud.disk.api..")
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
            .whereLayer("Api").mayOnlyBeAccessedByLayers("App","AllBundle","PlatForm")
            .whereLayer("Library").mayOnlyBeAccessedByLayers("App","AllBundle","PlatForm");
}
```

目前所有之前的架构设计约束已经完整解耦开，但由于测试文件和Hilt会生成一些编译的问题，所以我们还得新增配置文件进行过滤。

在resource文件夹中新增文件archunit\_ignore\_patterns.txt，新增过滤规则如下：

```text
.*com.cloud.disk.SmokeTesting.*
.*com.cloud.disk.DaggerSmokeTesting_*.*
```

运行架构守护测试命令如下：

```text
 ./gradlew app:testDebug --tests ArchRuleTest
```

我们可以看到已正常运行通过。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/8ac27e8110cf42699fc5ebd053fd7ea4~tplv-k3u1fbpfcp-zoom-1.image)

## 总结

经过不断的**演进重构**，目前我们终于把设计的架构守护测试通过，我们通过一张图来对比一下前面的区别。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b549536459bf4898a9da35682d467eb4~tplv-k3u1fbpfcp-zoom-1.image)

随着第一阶段的解耦重构完成，CloudDisk团队决定将团队划分为5个团队，分别管理文件、动态、用户中心、平台及公共库。整个项目不再统一使用一个Git大仓进行代码管理，每个团队能独立维护自己的代码仓库。

下一篇，移动应用遗留系统重构（11）- 制品管理篇将分享如何将解耦的模块进行二进制发布，模块间不再使用源码依赖编译，能按设计在独立的git仓库中进行管理。

