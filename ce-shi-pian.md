# 测试篇

## 前言

上一篇[移动应用遗留系统重构（5）- 重构方法篇](https://juejin.cn/post/6952298178095874055)我们分享了进行依赖解除的重构流程。主要为4个操作步骤，识别内聚包、解除依赖、移动、验收。同时最后也提出了一个问题，**重构时如何保证功能的正确性，不会修改出新问题？**

其实这个问题**容易但又不简单**。**容易**的是把修改得功能仔细测一篇保证所有功能正常就可以了。**不简单**的是如何全面、高效、可重复的执行这个过程。我们很容易联想到的方案就是**自动化测试**。但最大的问题是，对大部分遗留系统来说都是没有任何自动化测试。而且大量的坏味道代码，可测试性低，我们也很难补充充分的自动化测试。那么我们有什么折中的策略吗？

## 测试策略

我们先来看看Google Android开发者官网上对于测试的介绍，将不同的类型的测试分为三类测试（即小型、中型和大型测试）。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/8edc233a95b04711b5c33de2fe7f3689~tplv-k3u1fbpfcp-zoom-1.image)

> 图片来源[developer.android.com](https://developer.android.com/training/testing/fundamentals?hl=zh-cn)

* 小型测试是指单元测试，用于验证应用的行为，一次验证一个类。
* 中型测试是指集成测试，用于验证模块内堆栈级别之间的互动或相关模块之间的互动。
* 大型测试是指端到端测试，用于验证跨越了应用的多个模块的用户操作流程。

前面提到对于遗留单体系统来说通常没有任何自动化测试，并且通常内部结构耦合严重，所以实施中小型的成本非常高。**显然对于遗留系统，测试金字塔模型适用度较低。** 所以对于遗留系统，可能比较适合的策略模型如下：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ccd5e3b5cfea4b63abe2bc03e8ae7218~tplv-k3u1fbpfcp-zoom-1.image)

**对于遗留单体系统，一个可行的思路是先补充中大型的测试，作为基本的冒烟测试，重构优化内部结构后再及时补充中小型测试。**

## CloudDisk示例

对于我们这个浓缩版的CloudDisk，界面上也比较简单。主要是有一个主界面，主界面上主要为文件、动态、用户。（后续的MV\*重构篇会持续补充页面交互及逻辑）

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e15b2950a5214cc08d00294089207b12~tplv-k3u1fbpfcp-zoom-1.image) 我们可以设计一组UI的测试验证基本的功能。主要的几个测试点如下： 1. 主界面能正常运行并显示3个Fragment 2. 3个Fragment能正常显示 3. 点击登录按钮，能够跳转到登录页面

测试设计的用例如下：

```text
@RunWith(AndroidJUnit4.class)
@LargeTest
public class SmokeTesting {

    @Test
    public void should_show_fragment_list_when_activity_launch() {
        //given
        ActivityScenario<MainActivity> scenario = ActivityScenario.launch(MainActivity.class);
        scenario.onActivity(activity -> {
            //when
            onView(withText(R.string.tab_user)).perform(click());
            //then
            List<Fragment> fragments = activity.getSupportFragmentManager().getFragments();
            assertThat(fragments.size() == 3);
            assertThat(fragments.get(0) instanceof FileFragment);
            assertThat(fragments.get(1) instanceof DynamicFragment);
            assertThat(fragments.get(2) instanceof UserCenterFragment);
        });
    }

    @Test
    public void show_show_file_ui_when_click_tab_file() {
        //given
        ActivityScenario<MainActivity> scenario = ActivityScenario.launch(MainActivity.class);
        scenario.onActivity(activity -> {
            //when
            onView(withText(R.string.tab_file)).perform(click());
            //then
            onView(withText("Hello file fragment")).check(matches(isDisplayed()));
        });
    }

    @Test
    public void show_show_dynamic_ui_when_click_tab_dynamic() {
        //given
        ActivityScenario<MainActivity> scenario = ActivityScenario.launch(MainActivity.class);
        scenario.onActivity(activity -> {
            //when
            onView(withText(R.string.tab_dynamic)).perform(click());
            //then
            onView(withText("Hello dynamic fragment")).check(matches(withEffectiveVisibility(ViewMatchers.Visibility.VISIBLE)));
        });
    }

    @Test
    public void show_show_user_center_ui_when_click_tab_dynamic() {
        //given
        ActivityScenario<MainActivity> scenario = ActivityScenario.launch(MainActivity.class);
        scenario.onActivity(activity -> {
            //when
            onView(withText(R.string.tab_user)).perform(click());
            //then
            onView(withText("Hello user center fragment")).check(matches(withEffectiveVisibility(ViewMatchers.Visibility.VISIBLE)));
        });
    }

    @Test
    public void show_show_login_ui_when_click_login_button() {
        //given
        ActivityScenario<MainActivity> scenario = ActivityScenario.launch(MainActivity.class);
        scenario.onActivity(activity -> {
            Intents.init();
            //when
            onView(withId(R.id.fab)).perform(click());
            //then
            intended(IntentMatchers.hasComponent("com.cloud.disk.platform.login.LoginActivity"));
            Intents.release();
        });
    }
}
```

详细代码见[Github提交](https://github.com/junbin1011/CloudDisk/commit/e5a9757db372a01c22d433434dad2ee5d643fa55)

我们可以将用例运行在[Robolectric](http://robolectric.org/)上，提高反馈的速度，执行命令如下：

```text
./gradlew testDebug --tests SmokeTesting
```

测试执行结果如下：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ce5e71cb71f447d4a98b8a63749f19fc~tplv-k3u1fbpfcp-zoom-1.image)

> 当然实际的项目里情况更复杂，数据可能来自网络服务、数据库等等。我们还需要进行Mock。后续的MV\*重构篇会持续补充常见坏味道示例代码及更多的自动化测试用例。

更多测试框架及设计可以参考Google官方 [在 Android 平台上测试应用](https://developer.android.com/training/testing?hl=zh-cn)

## 总结

这一篇我们介绍了常用的测试分类及遗留系统的测试策略，对于遗留单体系统，一个可行的思路是先补充中大型的测试，作为基本的冒烟测试，重构优化内部结构后再及时补充中小型测试。同时也给CloudDisk补充了一组基础的大型测试作为冒烟测试，作为后续重构的基本守护测试。

下一篇移动应用遗留系统重构（7）- 解耦重构演示篇（一） 我们将基于方法篇的流程开始对CloudDisk进行重构的改造，具体的解耦操作会以视频的方式展示。

## 参考资料

[developer.android.com](https://developer.android.com/training/testing/fundamentals?hl=zh-cn)

