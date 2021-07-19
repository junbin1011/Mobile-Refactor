# 编译调试篇

## 前言

上一篇[移动应用遗留系统重构（11）- 制品管理篇](https://juejin.cn/post/6973836199655899149)我们完成通过本地maven进行二进制管理改造。在架构设计中App模块是整体工程的集成，用于进行最后的发布。业务Bundle在独立的仓库中维护，并且Bundle都是以Lib的形式存在，我们需要能够对各个业务Bundle进行独立的调试开发。

本篇我们将继续对CloudDisk进行改造，支持Bundle进行独立的调试。

## 调试方式

### 1. 支持Lib及Application切换

我们可以通过配置Gradle的形式，让Module支持Lib及Application的调试，这样既可以满足运行调试，也可以满足发布aar。但是缺点是Gradle要进行配置，并且调试代码和正式代码都在一个工程中。

我们以FileBundle为例，Gradle配置情况如下：

```text
//gradle 文件修改如下

def isApp = false
if (isApp) {
    //可运行
    apply plugin: 'com.android.application'
} else {
    //作为库
    apply plugin: 'com.android.library'
}

defaultConfig {
    if (isApp) {
        applicationId 'com.cloud.filebundle'
    }
}
```

在src下新增debug目录，这样当打debug包时，该目录下的代码也会一起打包，当打release包时，则不会带上。这样也可方便把测试代码区分开。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/799881281217424dbada7c577bdd27e9~tplv-k3u1fbpfcp-zoom-1.image)

我们直接把FileFragment在DebugActivity中显示出来，运行情况如下：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/1e6b29cfbd324f2186fefe1343b6a6d0~tplv-k3u1fbpfcp-zoom-1.image)

完整代码示例见[Github](https://github.com/junbin1011/CloudDisk/commit/7ed0b1e3eff0d40708bcd60e5d0a34cd2b8f1f00)

### 2. 新增Debug工程用于调试

接下来我们通过DynamicBundle进行演示，新增一个module用于专门调试,这种做法的好处是测试代码完全隔离。

新建名为DynamicDebug的Module，加上对于的依赖，如下：

```text
   implementation 'com.cloud.disk:api:1.0.0'
   implementation project(':dynamicBundle')
```

同样新增DebugActivity用于调试DynamicFragment。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/02f7540f2a7242489884ad1277b98b29~tplv-k3u1fbpfcp-zoom-1.image)

但运行调试发现编译出错，具体的日志如下：

```text
 error: [Dagger/MissingBinding] com.cloud.disk.api.file.TransferFile cannot be provided without an @Provides-annotated method.
  public abstract static class ApplicationC implements DebugApplication_GeneratedInjector,
                         ^
      com.cloud.disk.api.file.TransferFile is injected at
          com.cloud.disk.bundle.dynamic.DynamicFragment.transferFile
      com.cloud.disk.bundle.dynamic.DynamicFragment is injected at
          com.cloud.disk.bundle.dynamic.DynamicFragment_GeneratedInjector.injectDynamicFragment(com.cloud.disk.bundle.dynamic.DynamicFragment) [com.cloud.dynamicdebug.DebugApplication_HiltComponents.ApplicationC → com.cloud.dynamicdebug.DebugApplication_HiltComponents.ActivityRetainedC → com.cloud.dynamicdebug.DebugApplication_HiltComponents.ActivityC → com.cloud.dynamicdebug.DebugApplication_HiltComponents.FragmentC]
  It is also requested at:
      com.cloud.disk.bundle.dynamic.DynamicController.transferFile
```

因为DynamicBundle运行需要依赖TransferFile及UseState接口的实现，这个时候DynamicBundle我们独立调试，并没有将接口的实现加载进来。**这个时候我们建议的做法是采用Mock，不再把具体的实现加载进来。**

新增TransferFile及UseState的接口Mock如下：

```text
@Module
@InstallIn(ActivityComponent.class)
public abstract class MockFileModule {
    @Binds
    public abstract TransferFile bindTransferFile(
            MockTransferFileImpl transferFileImpl
    );
}

public class MockTransferFileImpl implements TransferFile {
    @Inject
    public MockTransferFileImpl() {
    }

    @Override
    public FileInfo upload(String s) {
        return new FileInfo();
    }

    @Override
    public FileInfo download(String s) {
        return new FileInfo();
    }
}
```

```text
@Module
@InstallIn(ActivityComponent.class)
public abstract class MockUserModule {
    @Binds
    public abstract UserState bindUserState(
            MockUserStateImpl userStateImpl
    );
}

public class MockUserStateImpl implements UserState {
    @Inject
    public MockUserStateImpl() {
    }

    @Override
    public String getUserId() {
        return "";
    }

    @Override
    public boolean isLogin() {
        return true;
    }
}
```

增加mock后，编译正常，运行结果如下：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/47f7a3119a094cec9baa81d3fc4bf2ff~tplv-k3u1fbpfcp-zoom-1.image)

完整代码示例见[Github](https://github.com/junbin1011/CloudDisk/commit/b868d2514aed69f1b6fab79e3a047ce8c5c63b2d)

### 3. 自动化测试

这是非常推荐的一种方式，可重复执行且反馈速度快。前面我们已经有一组冒烟测试的示例。接下来介绍对userBundle的UserCenterFragment进行测试，我们同样使用Robolectric提高反馈的速度。

Gradle增加依赖配置，如下：

```text
    testImplementation 'junit:junit:4.13.2'
    testImplementation "com.google.truth:truth:1.0.1"
    testImplementation 'com.tngtech.archunit:archunit-junit4:0.15.0'
    testImplementation 'org.robolectric:robolectric:4.5.1'
    testImplementation 'androidx.test:core:1.3.0'
    testImplementation 'androidx.test.ext:junit:1.1.2'
    testImplementation 'androidx.test.espresso:espresso-core:3.3.0'
    testImplementation 'androidx.test.espresso:espresso-intents:3.3.0'
    debugImplementation "androidx.fragment:fragment-testing:1.3.0"
```

编写测试用例如下：

```text
@RunWith(AndroidJUnit4.class)
@LargeTest
public class UserCenterFragmentTest {
    @Test
    public void show_show_user_center_ui_when_click_tab_dynamic() {
        //given
        FragmentScenario<UserCenterFragment> scenario = FragmentScenario.launchInContainer(UserCenterFragment.class);
        scenario.onFragment(activity -> {
            //then
            onView(withText("Hello user center fragment")).check(matches(isDisplayed()));
        });
    }
}
```

执行测试命令：

```text
 ./gradlew userBundle:testDebug
```

运行结果如下：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/bb457be8da764cb4b749325698adb947~tplv-k3u1fbpfcp-zoom-1.image)

完整代码示例见[Github](https://github.com/junbin1011/CloudDisk/commit/e8439d74c113e6ef73ee12bd8c9893a853525e2a)

## 总结

经过了解耦、分库及编译调试的优化，一段时间内，CloudDisk的开发效率得到了很大的提升。

但随着业务的演进，由于历史的原因，代码中存在很多Activity及Controller的上帝类。团队发现独立的业务Bundle质量还是比较差，Bug也比较多。团队决定再次对模块类的代码进行优化重构，采用新的MVP、MVVM架构将数据及视图分离。

下一篇，移动应用遗留系统重构（13）- MVP重构示例篇，一镜到底，我们将通过视频及文章的方式进行演示。

