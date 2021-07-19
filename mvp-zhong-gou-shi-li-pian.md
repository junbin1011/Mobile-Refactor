# MVP重构示例篇

## 前言

上一篇[移动应用遗留系统重构（12）- 编译调试篇](https://juejin.cn/post/6974634615537401886) 介绍，经过了解耦、分库及编译调试的优化，一段时间内，CloudDisk的开发效率得到了很大的提升。

但随着业务的演进，由于历史的原因，代码中存在很多Activity及Controller的上帝类。今天我们将拿File Bundle作为例子，为大家总结重构的流程，演示如何一步一步将上帝类重构为MVP架构。

**视频演示地址:** [https://mp.weixin.qq.com/s/zjeln\_eqAN45CDbSrWjLaA](https://mp.weixin.qq.com/s/zjeln_eqAN45CDbSrWjLaA)

## 重构流程

```text
graph TD
A(1.梳理业务逻辑)-->B(2.分析原有的代码设计)
B-->C(3.补充守护测试)
C-->D(4.简单设计)
D-->E(5.小步安全重构)
E-->F(6.集成验收测试)
```

### 1. 梳理业务逻辑

通过往往这一步是最难的。由于人员更迭、产品的迭代，给这一步带来很大的挑战。我们可以尝试从一下几方面来补全信息。

1. 找人：产品经理、设计人员、测试人员进行确认和答疑
2. 找文档：查看原有的需求文档、设计文档、测试用例、设计稿
3. 看代码：从原有的代码设计中去梳理业务

经过梳理确认，File Bundle的现有的业务如下：

```text
graph TD
A(进入文件页面)-->B(从网络加载文件列表数据)
B --> C{数据是否加载成功}
C -->|加载成功| D(显示文件列表-文件名和文件大小)
C -->|网络异常| E(显示NetworkErrorException)
C -->|数据为空| F(显示empty data)
E -->|点击触发刷新|B
F -->|点击触发刷新|B
```

> 文件大小转换为以B、K、M、G单位显示

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/9a3b64237ff544bf9b01d0d222bba3e3~tplv-k3u1fbpfcp-zoom-1.image)

### 2.分析原有的代码设计

我们以主要上帝类FileFragment为例，代码如下：

```text
@AndroidEntryPoint
public class FileFragment extends Fragment {

    @Inject
    FileController fileController;

    private RecyclerView fileListRecycleView;
    private TextView tvMessage;

    public static FileFragment newInstance() {
        FileFragment fragment = new FileFragment();
        Bundle args = new Bundle();
        fragment.setArguments(args);
        return fragment;
    }

    @Override
    public View onCreateView(LayoutInflater inflater, ViewGroup container,
                             Bundle savedInstanceState) {

        View view = inflater.inflate(R.layout.fragment_file, container, false);
        fileListRecycleView = view.findViewById(R.id.file_list);
        tvMessage = view.findViewById(R.id.tv_message);
        tvMessage.setOnClickListener(v -> getFileList());
        return view;
    }

    @Override
    public void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        getFileList();
    }

    private void getFileList() {
        new Thread(() -> {
            Message message = new Message();
            try {
                List<FileInfo> infoList = fileController.getFileList();
                message.what = 1;
                message.obj = infoList;
            } catch (NetworkErrorException e) {
                message.what = 0;
                message.obj = "NetworkErrorException";
                e.printStackTrace();
            }
            mHandler.sendMessage(message);
        }).start();
    }

    Handler mHandler = new Handler(new Handler.Callback() {
        @Override
        public boolean handleMessage(@NonNull Message msg) {
            if (msg.what == 1) {
                showTip(false);
                //显示网络数据
                List<FileInfo> infoList = (List<FileInfo>) msg.obj;
                FileListAdapter fileListAdapter = new FileListAdapter(infoList, getActivity());
                fileListRecycleView.addItemDecoration(new DividerItemDecoration(
                        getActivity(), DividerItemDecoration.VERTICAL));
                //设置布局显示格式
                fileListRecycleView.setLayoutManager(new LinearLayoutManager(getActivity()));
                fileListRecycleView.setAdapter(fileListAdapter);
            } else if (msg.what == 0) {
                showTip(true);
                //显示异常提醒数据
                tvMessage.setText(msg.obj.toString());
            } else {
                showTip(true);
                //显示空数据
                tvMessage.setText("empty data");
            }
            return false;
        }
    });

    public void showTip(boolean show) {
        if (show) {
            tvMessage.setVisibility(View.VISIBLE);
            fileListRecycleView.setVisibility(View.GONE);
        } else {
            tvMessage.setVisibility(View.GONE);
            fileListRecycleView.setVisibility(View.VISIBLE);
        }
    }
}
```

从代码中我们可以看到主要的一些设计问题，如下： 1. 主要的获取文件、异常逻辑判断、界面刷新控制都是在一个类里面，不利于后续的扩张及修改维护。我们希望类的职责更加单一，逻辑和视图能够进行分离。 2. 存在粗暴的new Thread进行管理 3. Handler 存在内存泄露风险 4. 存在规范问题，例如empty data字符串没有使用xml进行管理、代码中由无效的导包等 5. 代码中没有任何守护测试

完整的所有代码见[Github](https://github.com/junbin1011/CloudDisk/commit/3c4c4d98bcef46357f4c653431dc53e6d691e397)

### 3. 补充守护测试

参考重构篇中，我们制定的策略，我们可以先做大型的测试，做为守护测试。

```text
@RunWith(AndroidJUnit4.class)
@LargeTest
@HiltAndroidTest
@Config(application = HiltTestApplication.class)
public class FileFragmentTest {

    @Rule
    public HiltAndroidRule hiltRule = new HiltAndroidRule(this);

    @Test
    public void show_show_file_list_when_get_success() {
        //given
        ActivityScenario<DebugActivity> scenario = ActivityScenario.launch(DebugActivity.class);
        scenario.onActivity(activity -> {
            //then
            onView(withText("遗留代码重构.pdf")).check(matches(isDisplayed()));
            onView(withText("100.00K")).check(matches(isDisplayed()));
            onView(withText("系统组件化.pdf")).check(matches(isDisplayed()));
            onView(withText("9.67K")).check(matches(isDisplayed()));
        });
    }
}
```

**我们发现这个用不会通过，我们主要面临一下2个问题。**

1. 网络请求是异步的，异步逻辑可能在测试执行完还没有触发到
2. 网络数据是动态的，我们断言的数据没办法确定

所以我们采用的做法就是Mock，我不稳定的依赖Mock掉。我们同样使用Shadow来进行多获取网络数据方法进行Mock，代码如下：

```text
@Implements(FileFragment.class)
public class ShadowFileFragment {

    @RealObject
    public FileFragment fileFragment;

    enum State {
        SUCCESS,
        ERROR,
        EMPTY
    }

    public static State state = State.SUCCESS;

    @Implementation
    protected void getFileList() {
        System.out.println("shadow .... .....");
        Message message = new Message();
        if (state == State.SUCCESS) {
            ArrayList<FileInfo> infoList = new ArrayList<>();
            infoList.add(new FileInfo("遗留代码重构.pdf", 102400));
            infoList.add(new FileInfo("系统组件化.pdf", 9900));
            message.what = 1;
            message.obj = infoList;
        } else if (state == State.ERROR) {
            message.what = 0;
            message.obj = "NetworkErrorException";
        } else if (state == State.EMPTY) {
            message.what = 1;
            message.obj = null;
        }
        fileFragment.mHandler.sendMessage(message);
    }
}
```

调整后的测试用例如下：

```text
 @Test
    public void show_show_file_list_when_get_success() {
        //given
        ShadowFileFragment.state = ShadowFileFragment.State.SUCCESS;
        //when
        ActivityScenario<DebugActivity> scenario = ActivityScenario.launch(DebugActivity.class);
        scenario.onActivity(activity -> {
            //then
            onView(withText("遗留代码重构.pdf")).check(matches(isDisplayed()));
            onView(withText("100.00K")).check(matches(isDisplayed()));
            onView(withText("系统组件化.pdf")).check(matches(isDisplayed()));
            onView(withText("9.67K")).check(matches(isDisplayed()));
        });
    }

    @Test
    public void show_show_error_tip_when_net_work_exception() {
        //given
        ShadowFileFragment.state = ShadowFileFragment.State.ERROR;
        //when
        ActivityScenario<DebugActivity> scenario = ActivityScenario.launch(DebugActivity.class);
        scenario.onActivity(activity -> {
            //then
            onView(withText("NetworkErrorException")).check(matches(isDisplayed()));
        });
    }

    @Test
    public void show_show_empty_tip_when_not_has_data() {
        //given
        ShadowFileFragment.state = ShadowFileFragment.State.EMPTY;
        //when
        ActivityScenario<DebugActivity> scenario = ActivityScenario.launch(DebugActivity.class);
        scenario.onActivity(activity -> {
            //then
            onView(withText("empty data")).check(matches(isDisplayed()));
        });
    }
```

中间我们用允许测试脚步时发现，之前的旧代码还有一处逻辑的错误。数据为空的判断应该加在网络数据回调中。

```text
Caused by: java.lang.NullPointerException
    at com.cloud.disk.bundle.file.FileFragment$1.handleMessage(FileFragment.java:96)
    at android.os.Handler.dispatchMessage(Handler.java:102)
```

我们同步将异常逻辑进行修改，如下：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/035a8a19ed9347289437f5416caa5297~tplv-k3u1fbpfcp-zoom-1.image)

最后我们完成了基本的守护测试，运行结果如下：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ada7610b565544a8ab7cf854945e146c~tplv-k3u1fbpfcp-zoom-1.image)

完整的所有代码见[Github](https://github.com/junbin1011/CloudDisk/commit/3c4c4d98bcef46357f4c653431dc53e6d691e397)

### 4. 简单设计

#### MVP架构

```text
graph TD
B(Presenter)-->A(Model)
B-->|interface|C(View)
C-->|interface|B
```

1. 业务逻辑和视图分离
2. Presenter和View之间通过接口交互
3. 为了更高效管理线程，团队决定使用Rxjava进行线程统一管理。架构风格参考[architecture-samples](https://github.com/android/architecture-samples)

#### 接口设计

1. Contract接口设计

```text
public interface FileListContract {

    interface View  {
        showFileList(List<FileInfo> fileList);
        showNetWorkException(String errorMessage);
        showEmptyData();
    }

    interface Presenter {

        void getFileList();
    }
}
```

1. 数据接口设计

```text
public interface FileDataSource {
     Flowable<List<FileInfo>> getFileList();
}
```

### 5.小步安全重构

* 抽取FileFragment的业务逻辑到FilePresenter
* FileFragment 提取UI接口
* 抽取FileDataSource

> 重构手法包含提取接口、移动方法、移动类、抽取方法、内联、提取变量等等。**过程还需要根据重构适当调整测试用例。**

详细的演示见[视频](https://mp.weixin.qq.com/s/zjeln_eqAN45CDbSrWjLaA)

* 补充测试用例
* 补充FileUtils计算文件大小测试

```text
@RunWith(JUnit4.class)
@SmallTest
public class FileUtilsTest {

    @Test
    public void should_return_B_unit_when_file_size_in_its_range() {
        //given
        long fileSize = 100;
        //when
        String format = FileUtils.formatFileSize(fileSize);
        //then
        assertEquals("100.00B", format);
    }

    @Test
    public void should_return_K_unit_when_file_size_in_its_range() {
        //given
        long fileSize = 1034;
        //when
        String format = FileUtils.formatFileSize(fileSize);
        //then
        assertEquals("1.01K", format);
    }

    @Test
    public void should_return_M_unit_when_file_size_in_its_range() {
        //given
        long fileSize = 1084000;
        //when
        String format = FileUtils.formatFileSize(fileSize);
        //then
        assertEquals("1.03M", format);
    }

    @Test
    public void should_return_G_unit_when_file_size_in_its_range() {
        //given
        long fileSize = 1114000000;
        //when
        String format = FileUtils.formatFileSize(fileSize);
        //then
        assertEquals("1.04G", format);
    }
}
```

1. 补充Presenter业务逻辑测试

```text
@RunWith(JUnit4.class)
@MediumTest
public class FilePresenterImplTest {

    @Rule
    public RxSchedulerRule rule = new RxSchedulerRule();

    @Test
    public void should_return_file_list_when_call_data_source_success() throws NetworkErrorException {
        //given
        FileListContract.View mockView = mock(FileListContract.View.class);
        FileDataSource mockFileDataSource = mock(FileDataSource.class);
        List<FileInfo> fileList = new ArrayList<>();
        fileList.add(new FileInfo("遗留代码重构.pdf", 102400));
        fileList.add(new FileInfo("系统组件化.pdf", 9900));
        when(mockFileDataSource.getFileList()).thenReturn(Flowable.fromArray(fileList));
        FileListContract.FilePresenter filePresenter = new FilePresenterImpl(mockFileDataSource, mockView);
        //when
        filePresenter.getFileList();
        //then
        verify(mockView).showFileList(anyList());
    }

    @Test
    public void should_show_empty_data_when_call_data_source_return_empty() throws NetworkErrorException {
        //given
        FileListContract.View mockView = mock(FileListContract.View.class);
        FileDataSource mockFileDataSource = mock(FileDataSource.class);
        when(mockFileDataSource.getFileList()).thenReturn(Flowable.fromArray(new ArrayList<>()));
        FileListContract.FilePresenter filePresenter = new FilePresenterImpl(mockFileDataSource, mockView);
        //when
        filePresenter.getFileList();
        //then
        verify(mockView).showEmptyData();
    }

    @Test
    public void should_show_network_exception_when_call_data_source_return_net_error() throws NetworkErrorException {
        //given
        FileListContract.View mockView = mock(FileListContract.View.class);
        FileDataSource mockFileDataSource = mock(FileDataSource.class);
        when(mockFileDataSource.getFileList()).thenThrow(new NetworkErrorException());
        FileListContract.FilePresenter filePresenter = new FilePresenterImpl(mockFileDataSource, mockView);
        //when
        filePresenter.getFileList();
        //then
        verify(mockView).showNetWorkException("NetworkErrorException");
    }
}
```

我们可以通过查看Coverage查看逻辑的覆盖情况，判断是否有重要的逻辑遗漏。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/6b5ff0f4675841c899473f5f0c4ee025~tplv-k3u1fbpfcp-zoom-1.image)

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a1b81a61c179407b96eb0b41989bdfb1~tplv-k3u1fbpfcp-zoom-1.image)

最后我们运行file bundle所有的测试，报告如下：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e195ae6e3b0f426eaa2c6d2d8a3d1e3d~tplv-k3u1fbpfcp-zoom-1.image)

### 6.集成验收测试

1. file bundle模块发布1.0.1版本

```text
implementation 'com.cloud.disk.bundle:file:1.0.1'
```

1. 执行守护测试

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/939949c16c824747bdb0bade6e6d9068~tplv-k3u1fbpfcp-zoom-1.image)

1. 运行检查

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d0846861f29149ad990a2c1d951be2a6~tplv-k3u1fbpfcp-zoom-1.image)

## 总结

本篇介绍了文件模块团队将文件主页重构为MVP架构，并且补充了自动化测试。经过重构后，团队的开发效率和版本质量有了明显的提升。有了文件模块的打样，给其他团队带来了很大的信心。

接下来，我们将继续分享动态模块的重构之旅。与文件模块不一样的是动态模块决定采用Kotlin+MVVM架构。

下一篇，单体移动应用“模块化”演进之旅（14）- Kotlin+MVVM重构示例篇

