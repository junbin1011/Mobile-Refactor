# Kotlin+MVVM重构示例篇

## 前言

上一篇[移动应用遗留系统重构（13）-MVP重构示例篇](https://juejin.cn/post/6975877150314332168)介绍了文件模块团队将文件主页重构为MVP架构，并且补充了自动化测试。经过重构后，团队的开发效率和版本质量有了明显的提升。动态模块的业务比文件模块更复杂，并且这次团队决定使用新的开发语言Kotlin及MVVM架构。

本篇我们将拿DynamicBundle作为例子，为大家继续演示如何从Java代码过渡为Kotlin代码，以及如何一步一步将上帝类重构演化为MVVM架构。

**视频演示地址:** [https://mp.weixin.qq.com/s/vex4Kn6Ts-dI3KX2Yiuq6w](https://mp.weixin.qq.com/s/vex4Kn6Ts-dI3KX2Yiuq6w)

## 重构流程

从新回忆下上一篇我们分析的重构流程，对于转Kotlin语言，我们建议也做完至第3步，有了守护测试再进行转换，这样更加安全。流程如下：

```text
graph TD
A(1.梳理业务逻辑)-->B(2.分析原有的代码设计)
B-->C(3.补充守护测试)
C-->D(4.简单设计)
D-->E(5.小步安全重构)
E-->F(6.集成验收测试)
```

### 1. 梳理业务逻辑

上篇提到我们可以尝试从一下几方面来补全信息。

1. 找人：产品经理、设计人员、测试人员进行确认和答疑
2. 找文档：查看原有的需求文档、设计文档、测试用例、设计稿
3. 看代码：从原有的代码设计中去梳理业务

经过梳理确认，FileBundle的现有的业务如下：

```text
graph TD
A(进入动态页面)-->B(从网络加载动态列表数据)
B --> C{数据是否加载成功}
C -->|加载成功| D(显示动态列表-内容和日期)
C -->|网络异常| E{是否存在本地缓存数据}
C -->|数据为空| F(显示empty data)
E -->|存在缓存数据|D
E -->|不存在缓存数据|G(显示NetworkErrorException)
F -->|点击触发刷新|B
G -->|点击触发刷新|B
```

> 时间转换为 yyyy-MM-dd HH:mm:ss 显示

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/dceb125078de4726a486c6ae850bfb57~tplv-k3u1fbpfcp-zoom-1.image)

### 2.分析原有的代码设计

我们以主要上帝类DynamicFragment为例，代码如下：

```text
@Route(path = "/dynamicBundle/dynamic")
@AndroidEntryPoint
public class DynamicFragment extends Fragment {

    @Inject
    DynamicController dynamicController;
    Button btnUpload;
    @Inject
    TransferFile transferFile;

    private RecyclerView dynamicListRecycleView;
    private TextView tvMessage;

    public static DynamicFragment newInstance() {
        DynamicFragment fragment = new DynamicFragment();
        Bundle args = new Bundle();
        fragment.setArguments(args);
        return fragment;
    }

    @Override
    public void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);

    }

    @Override
    public View onCreateView(LayoutInflater inflater, ViewGroup container,
                             Bundle savedInstanceState) {
        View view = inflater.inflate(R.layout.fragment_dynamic, container, false);
        btnUpload = view.findViewById(R.id.btn_upload);
        btnUpload.setOnClickListener(v -> uploadDynamic());
        dynamicListRecycleView = view.findViewById(R.id.file_list);
        tvMessage = view.findViewById(R.id.tv_message);
        tvMessage.setOnClickListener(v -> getDynamicList());
        getDynamicList();
        return view;
    }

    public void uploadDynamic() {
        //上传文件
        FileInfo fileInfo = transferFile.upload("/data/data/user.png");
        dynamicController.post(new Dynamic(0, "第一个动态", System.currentTimeMillis()), fileInfo);
    }

    public void getDynamicList() {
        new Thread(() -> {
            Message message = new Message();
            try {
                List<Dynamic> dynamicList = dynamicController.getDynamicList();
                message.what = 1;
                message.obj = dynamicList;
            } catch (NetworkErrorException e) {
                message.what = 0;
                message.obj = "NetworkErrorException";
                e.printStackTrace();
            }
            mHandler.sendMessage(message);
        }).start();
    }

    public Handler mHandler = new Handler(new Handler.Callback() {
        @Override
        public boolean handleMessage(@NonNull Message msg) {
            if (msg.what == 1) {
                showTip(false);
                //显示网络数据
                List<Dynamic> dynamicList = (List<Dynamic>) msg.obj;
                if (dynamicList == null || dynamicList.size() == 0) {
                    showTip(true);
                    //显示空数据
                    tvMessage.setText("empty data");

                } else {
                    DynamicListAdapter fileListAdapter = new DynamicListAdapter(dynamicList, getActivity());
                    dynamicListRecycleView.addItemDecoration(new DividerItemDecoration(
                            getActivity(), DividerItemDecoration.VERTICAL));
                    //设置布局显示格式
                    dynamicListRecycleView.setLayoutManager(new LinearLayoutManager(getActivity()));
                    dynamicListRecycleView.setAdapter(fileListAdapter);
                    //从网络中更新到数据保持到缓存之中
                    dynamicController.saveDynamicToCache(dynamicList);
                }
            } else if (msg.what == 0) {
                //尝试从缓存中读取数据
                List<Dynamic> dynamicList = dynamicController.getDynamicListFromCache();
                if (dynamicList == null || dynamicList.size() == 0) {
                    showTip(true);
                    //显示异常提醒数据
                    tvMessage.setText(msg.obj.toString());
                } else {
                    DynamicListAdapter fileListAdapter = new DynamicListAdapter(dynamicList, getActivity());
                    dynamicListRecycleView.addItemDecoration(new DividerItemDecoration(
                            getActivity(), DividerItemDecoration.VERTICAL));
                    //设置布局显示格式
                    dynamicListRecycleView.setLayoutManager(new LinearLayoutManager(getActivity()));
                    dynamicListRecycleView.setAdapter(fileListAdapter);
                }

            }
            return false;
        }
    });

    public void showTip(boolean show) {
        if (show) {
            tvMessage.setVisibility(View.VISIBLE);
            dynamicListRecycleView.setVisibility(View.GONE);
        } else {
            tvMessage.setVisibility(View.GONE);
            dynamicListRecycleView.setVisibility(View.VISIBLE);
        }
    }
}
```

从代码中我们可以看到主要的一些设计问题，如下： 1. 主要的获取动态列表、异常逻辑判断、数据缓存判断、界面刷新控制都是在一个类里面，不利于后续的扩张及修改维护。我们希望类的职责更加单一，逻辑和视图能够进行分离。 2. 存在粗暴的new Thread进行管理 3. Handler 存在内存泄露风险 4. 存在重复代码，例如列表数据的展示 5. 代码中没有任何守护测试

完整的所有代码见[Github](https://github.com/junbin1011/CloudDisk/commit/dd2abeb9000d81af340fbface4f25182ada922f0)

### 3. 补充守护测试

参考重构篇中，我们制定的策略，我们可以先做大型的测试，做为守护测试。同样我们会将数据库和网络相关的操作进行mock。测试用例主要包含梳理出来的主要业务逻辑，包括正常显示数据、网络异常存在缓存数据、网络异常没有缓存数据、空数据。代码如下：

```text
@RunWith(AndroidJUnit4::class)
@LargeTest
@HiltAndroidTest
@Config(application = HiltTestApplication::class, shadows = [ShadowDynamicFragment::class, ShadowDynamicController::class])
class DynamicFragmentTest {
    @get:Rule
    var hiltRule = HiltAndroidRule(this)

    @Test
    fun `show show dynamic list when get success`() {
        //given
        ShadowDynamicFragment.state = ShadowDynamicFragment.State.SUCCESS
        //when
        val scenario: ActivityScenario<DebugActivity> = ActivityScenario.launch(DebugActivity::class.java)
        scenario.onActivity {
            //then
            onView(withText("今天天气真不错！")).check(matches(isDisplayed()))
            onView(withText("2021-03-17 14:47:55")).check(matches(isDisplayed()))
            onView(withText("这个连续剧值得追！")).check(matches(isDisplayed()))
            onView(withText("2021-03-17 14:48:08")).check(matches(isDisplayed()))
        }
    }

    @Test
    fun `show show dynamic list when net work exception but have cache`() {
        //given
        ShadowDynamicFragment.state = ShadowDynamicFragment.State.ERROR
        ShadowDynamicController.state = ShadowDynamicController.State.DATA
        //when
        val scenario: ActivityScenario<DebugActivity> = ActivityScenario.launch(DebugActivity::class.java)
        scenario.onActivity {
            //then
            onView(withText("今天天气真不错！")).check(matches(isDisplayed()))
            onView(withText("2021-03-17 14:47:55")).check(matches(isDisplayed()))
            onView(withText("这个连续剧值得追！")).check(matches(isDisplayed()))
            onView(withText("2021-03-17 14:48:08")).check(matches(isDisplayed()))
        }
    }

    @Test
    fun `show show error tip when net work exception and not have cache`() {
        //given
        ShadowDynamicFragment.state = ShadowDynamicFragment.State.ERROR
        ShadowDynamicController.state = ShadowDynamicController.State.EMPTY
        //when
        val scenario: ActivityScenario<DebugActivity> = ActivityScenario.launch(DebugActivity::class.java)
        scenario.onActivity {
            //then
            onView(withText("NetworkErrorException")).check(matches(isDisplayed()))
        }
    }

    @Test
    fun `show show empty tip when not has data`() {
        //given
        ShadowDynamicFragment.state = ShadowDynamicFragment.State.EMPTY
        //when
        val scenario: ActivityScenario<DebugActivity> = ActivityScenario.launch(DebugActivity::class.java)
        scenario.onActivity {
            //then
            onView(withText("empty data")).check(matches(isDisplayed()))
        }
    }
}
```

最后我们完成了基本的守护测试，运行结果如下：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/fde393487aac4e369c748aed64fb19d1~tplv-k3u1fbpfcp-zoom-1.image)

完整的所有代码见[Github](https://github.com/junbin1011/CloudDisk/commit/b9505f96999aa24b7b82b9deedc1763e332bf4de)

### [Kotlin](https://www.kotlincn.net/docs/reference/)

1. Kotlin与Java支持混编，我们可以将新增的代码用Kotlin编写。详细看Kotlin官网的[Java互操作](https://www.kotlincn.net/docs/reference/java-interop.html)
2. AndroidStudio支持将原有的Java代码转换为Kotlin代码 ![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/60b75973de0f4402b6d1693a6e66aa0a~tplv-k3u1fbpfcp-zoom-1.image)
3. AndroidStudio支持将查看Kotlin的字节码，Decompile为Java代码，方便理解一些语法特性

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/910370693a65454eb072b5cf3da0ac71~tplv-k3u1fbpfcp-zoom-1.image)

Dynamic模块团队成员决定使用2方案，将动态主页面转换为Kotlin代码。

> 转化过程中，尽量一次转换一个相对内聚的包，小步转换加测试验证，转换后再人工进行Review和调整

通过转换后，目前DynamicFragment的代码如下：

```text
@Route(path = "/dynamicBundle/dynamic")
@AndroidEntryPoint
class DynamicFragment : Fragment() {
    @Inject
    lateinit var dynamicController: DynamicController

    @Inject
    lateinit var transferFile: TransferFile

    private lateinit var btnUpload: Button
    private lateinit var dynamicListRecycleView: RecyclerView
    private lateinit var tvMessage: TextView


    override fun onCreateView(inflater: LayoutInflater, container: ViewGroup?,
                              savedInstanceState: Bundle?): View? {
        val view = inflater.inflate(R.layout.fragment_dynamic, container, false)
        btnUpload = view.findViewById(R.id.btn_upload)
        btnUpload.setOnClickListener { uploadDynamic() }
        dynamicListRecycleView = view.findViewById(R.id.file_list)
        tvMessage = view.findViewById(R.id.tv_message)
        tvMessage.setOnClickListener { getDynamicList() }
        getDynamicList()
        return view
    }

    private fun uploadDynamic() {
        //上传文件
        val fileInfo = transferFile.upload("/data/data/user.png")
        fileInfo?.let { dynamicController.post(Dynamic(0, "第一个动态", System.currentTimeMillis()), it) }
    }

    public fun getDynamicList() {
        Thread {
            val message = Message()
            val dynamicList = dynamicController.getDynamicList()
            message.what = 1
            message.obj = dynamicList
            mHandler.sendMessage(message)
        }.start()
    }

    var mHandler = Handler { msg ->
        if (msg.what == 1) {
            showTip(false)
            //显示网络数据
            var dynamicList = mutableListOf<Dynamic>()
            msg.obj?.let { dynamicList = msg.obj as MutableList<Dynamic> }
            if (dynamicList.isEmpty()) {
                showTip(true)
                //显示空数据
                tvMessage.text = "empty data"
            } else {
                val fileListAdapter = activity?.let { DynamicListAdapter(dynamicList, it) }
                dynamicListRecycleView.addItemDecoration(DividerItemDecoration(
                        activity, DividerItemDecoration.VERTICAL))
                //设置布局显示格式
                dynamicListRecycleView.layoutManager = LinearLayoutManager(activity)
                dynamicListRecycleView.adapter = fileListAdapter
                //从网络中更新到数据保持到缓存之中
                dynamicController.saveDynamicToCache(dynamicList)
            }
        } else if (msg.what == 0) {
            //尝试从缓存中读取数据
            val dynamicList = dynamicController.getDynamicListFromCache()
            if (dynamicList.isEmpty()) {
                showTip(true)
                //显示异常提醒数据
                tvMessage.text = msg.obj.toString()
            } else {
                val fileListAdapter = activity?.let { DynamicListAdapter(dynamicList, it) }
                dynamicListRecycleView.addItemDecoration(DividerItemDecoration(
                        activity, DividerItemDecoration.VERTICAL))
                //设置布局显示格式
                dynamicListRecycleView.layoutManager = LinearLayoutManager(activity)
                dynamicListRecycleView.adapter = fileListAdapter
            }
        }
        false
    }

    fun showTip(show: Boolean) {
        if (show) {
            tvMessage.visibility = View.VISIBLE
            dynamicListRecycleView.visibility = View.GONE
        } else {
            tvMessage.visibility = View.GONE
            dynamicListRecycleView.visibility = View.VISIBLE
        }
    }

    companion object {
        fun newInstance(): DynamicFragment {
            val fragment = DynamicFragment()
            val args = Bundle()
            fragment.arguments = args
            return fragment
        }
    }
}
```

### 4. 简单设计

#### MVVM架构

```text
graph TD
B(MVVM)-->A(Model)
B-->|bingding|C(View)
C-->|bingding|B
```

1. 业务逻辑和视图分离
2. ViewModel和View之间通过binding绑定，不用定义大量的接口
3. 为了更高效管理线程，团队决定使用coroutine进行线程统一管理。架构风格参考[architecture-samples](https://github.com/android/architecture-samples)

#### LiveData数据设计

```text
// 数据列表
 val dynamicListLiveData: LiveData<List<Dynamic>>
// 异常信息
 val errorMessageLiveData: LiveData<String>
```

#### 相关的第三方库

```text
implementation 'androidx.core:core-ktx:1.3.2'
implementation 'androidx.lifecycle:lifecycle-livedata-ktx:2.3.0'
implementation 'androidx.lifecycle:lifecycle-viewmodel-ktx:2.3.0'
implementation 'org.jetbrains.kotlinx:kotlinx-coroutines-core:1.4.1'
implementation 'org.jetbrains.kotlinx:kotlinx-coroutines-android:1.4.1'
```

### 5.小步安全重构

* 抽取DynamicFragment的业务逻辑到DynamicViewModel
* 定义LiveData数据及xml的绑定
* 抽取DymaicRepository
* 抽取DataSource接口

> 重构手法包含提取接口、移动方法、移动类、抽取方法、内联、提取变量等等。**Kotlin目前IDE不支持移动方法到类**

详细的演示见[视频](https://mp.weixin.qq.com/s/vex4Kn6Ts-dI3KX2Yiuq6w)

详细的代码见[Github](https://github.com/junbin1011/CloudDisk/commit/911a15c493d01dceb8ee56a0bba48fee6ad9847f)

#### 补充测试用例

1. 补充DateUtil计算日期测试

```text
@SmallTest
class DateUtilTest {

    @Test
    fun `should return 2021-03-17 14 47 5 when input is 1615963675000L`() {
        val format = DateUtil.getDateToString(1615963675000L)
        Assert.assertEquals("2021-03-17 14:47:55", format)
    }
}
```

1. 补充MVVM业务逻辑测试

```text
@ExperimentalCoroutinesApi
@SmallTest
class DynamicViewModelTest {
    private val testDispatcher = TestCoroutineDispatcher()

    @get:Rule
    val rule = InstantTaskExecutorRule()

    @Before
    fun setUp() {
        Dispatchers.setMain(testDispatcher)
    }

    @After
    fun tearDown() {
        Dispatchers.resetMain()
        testDispatcher.cleanupTestCoroutines()
    }

    @Test
    fun `show show dynamic list when get success`() = runBlocking {
        //given
        val mockTransferFile = mock(TransferFile::class.java)
        val mockDynamicRepository = mock(DynamicRepository::class.java)
        `when`(mockDynamicRepository.getDynamicList()).thenReturn(getMockData())
        val dynamicViewModel = DynamicViewModel(mockTransferFile, mockDynamicRepository)
        //when
        dynamicViewModel.getDynamicList()
        //then
        val dynamicOne = LiveDataTestUtil.getValue(dynamicViewModel.dynamicListLiveData)[0]
        assertThat(dynamicOne.id).isEqualTo(1)
        assertThat(dynamicOne.content).isEqualTo("今天天气真不错！")
        assertThat(dynamicOne.date).isEqualTo(1615963675000L)
        val dynamicTwo = LiveDataTestUtil.getValue(dynamicViewModel.dynamicListLiveData)[1]
        assertThat(dynamicTwo.id).isEqualTo(2)
        assertThat(dynamicTwo.content).isEqualTo("这个连续剧值得追！")
        assertThat(dynamicTwo.date).isEqualTo(1615963688000L)

    }

    @Test
    fun `show show dynamic list when net work exception but have cache`() = runBlocking {
        //given
        val mockTransferFile = mock(TransferFile::class.java)
        val mockDynamicRepository = mock(DynamicRepository::class.java)
        `when`(mockDynamicRepository.getDynamicList()).thenThrow(NetWorkErrorException::class.java)
        `when`(mockDynamicRepository.getDynamicListFromCache()).thenReturn(getMockData())
        val dynamicViewModel = DynamicViewModel(mockTransferFile, mockDynamicRepository)

        //when
        dynamicViewModel.getDynamicList()
        //then
        val dynamicOne = LiveDataTestUtil.getValue(dynamicViewModel.dynamicListLiveData)[0]
        assertThat(dynamicOne.id).isEqualTo(1)
        assertThat(dynamicOne.content).isEqualTo("今天天气真不错！")
        assertThat(dynamicOne.date).isEqualTo(1615963675000L)
        val dynamicTwo = LiveDataTestUtil.getValue(dynamicViewModel.dynamicListLiveData)[1]
        assertThat(dynamicTwo.id).isEqualTo(2)
        assertThat(dynamicTwo.content).isEqualTo("这个连续剧值得追！")
        assertThat(dynamicTwo.date).isEqualTo(1615963688000L)

    }

    @Test
    fun `show show error tip when net work exception and not have cache`() = runBlocking {
        //given
        val mockTransferFile = mock(TransferFile::class.java)
        val mockDynamicRepository = mock(DynamicRepository::class.java)
        `when`(mockDynamicRepository.getDynamicList()).thenThrow(NetWorkErrorException::class.java)
        val dynamicViewModel = DynamicViewModel(mockTransferFile, mockDynamicRepository)

        //when
        dynamicViewModel.getDynamicList()
        //then
        val errorMessage = LiveDataTestUtil.getValue(dynamicViewModel.errorMessageLiveData)
        assertThat(errorMessage).isEqualTo("NetWorkErrorException")
        val dynamicList = LiveDataTestUtil.getValue(dynamicViewModel.dynamicListLiveData)
        assertThat(dynamicList).isNull()
    }

    @Test
    fun `show show empty tip when not has data`() = runBlocking {
        //given
        val mockTransferFile = mock(TransferFile::class.java)
        val mockDynamicRepository = mock(DynamicRepository::class.java)
        `when`(mockDynamicRepository.getDynamicList()).thenReturn(null)
        val dynamicViewModel = DynamicViewModel(mockTransferFile, mockDynamicRepository)

        //when
        dynamicViewModel.getDynamicList()
        //then
        val dynamicList = LiveDataTestUtil.getValue(dynamicViewModel.dynamicListLiveData)
        assertThat(dynamicList).isNull()
    }

    private fun getMockData(): ArrayList<Dynamic> {
        val dynamicList = ArrayList<Dynamic>()
        dynamicList.add(Dynamic(1, "今天天气真不错！", 1615963675000L))
        dynamicList.add(Dynamic(2, "这个连续剧值得追！", 1615963688000L))
        return dynamicList
    }
}
```

dynamic bundle所有的测试，报告如下：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/1a4c0102904d4f2ea4ba11ce303f8259~tplv-k3u1fbpfcp-zoom-1.image)

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/42fae3f77869418383796f84162fd95c~tplv-k3u1fbpfcp-zoom-1.image)

#### DataBinging

Fragment 改造如下：

```text
 @Route(path = "/dynamicBundle/dynamic")
@AndroidEntryPoint
class DynamicFragment : Fragment() {

    private lateinit var binding: FragmentDynamicBinding
    private val dynamicViewModel: DynamicViewModel by viewModels()


    override fun onCreateView(inflater: LayoutInflater, container: ViewGroup?,
                              savedInstanceState: Bundle?): View? {
        binding = FragmentDynamicBinding.inflate(inflater, container, false)
        binding.btnUpload.setOnClickListener { dynamicViewModel.uploadDynamic() }
        binding.tvMessage.setOnClickListener { dynamicViewModel.getDynamicList() }
        dynamicViewModel.getDynamicList()
        subscribeUi()
        return binding.root
    }

    private fun subscribeUi() {
        dynamicViewModel.dynamicListLiveData.observe(viewLifecycleOwner, {
            if (it.isNullOrEmpty()) {
                showEmptyData()
            } else {
                showDynamicList(it)
            }
        })
        dynamicViewModel.errorMessageLiveData.observe(viewLifecycleOwner, {
            showErrorMessage(it)
        })
    }


    private fun showErrorMessage(errorMessage: String) {
        binding.showTip = true
        //显示异常提醒数据
        binding.tvMessage.text = errorMessage
    }

    private fun showDynamicList(dynamicList: List<Dynamic>) {
        binding.showTip = false
        val dynamicListAdapter = DynamicListAdapter()
        dynamicListAdapter.submitList(dynamicList)
        binding.dynamicList.addItemDecoration(DividerItemDecoration(
                activity, DividerItemDecoration.VERTICAL))
        //设置布局显示格式
        binding.dynamicList.layoutManager = LinearLayoutManager(activity)
        binding.dynamicList.adapter = dynamicListAdapter
    }

    private fun showEmptyData() {
        binding.showTip = true
        //显示空数据
        binding.tvMessage.text = "empty data"
    }

    companion object {
        fun newInstance(): DynamicFragment {
            val fragment = DynamicFragment()
            val args = Bundle()
            fragment.arguments = args
            return fragment
        }
    }
}

<?xml version="1.0" encoding="utf-8"?>
<layout xmlns:android="http://schemas.android.com/apk/res/android">

    <data>

        <import type="android.view.View" />

        <variable
            name="showTip"
            type="boolean" />

    </data>

    <FrameLayout
        android:layout_width="match_parent"
        android:layout_height="match_parent">

        <TextView
            android:id="@+id/tv_message"
            android:layout_width="match_parent"
            android:layout_height="match_parent"
            android:gravity="center"
            android:text=""
            android:visibility="@{showTip ? View.VISIBLE:View.GONE}" />

        <androidx.recyclerview.widget.RecyclerView
            android:id="@+id/dynamicList"
            android:layout_width="match_parent"
            android:layout_height="match_parent"
            android:visibility="@{showTip ? View.GONE:View.VISIBLE}" />

        <Button
            android:id="@+id/btn_upload"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:layout_gravity="right|bottom"
            android:layout_margin="10dp"
            android:text="upload" />
    </FrameLayout>

</layout>
```

Adapter 改造如下：

```text
 class DynamicListAdapter() : ListAdapter<Dynamic, DynamicListAdapter.DynamicVH>(DynamicDiffCallback()) {

    override fun onCreateViewHolder(parent: ViewGroup, viewType: Int): DynamicVH {
        return DynamicVH(DataBindingUtil.inflate(LayoutInflater.from(parent.context), R.layout.dynamic_list_item, parent, false))
    }

    override fun onBindViewHolder(holder: DynamicVH, position: Int) {
        holder.binding.dynamic = getItem(position)
        holder.binding.executePendingBindings()
    }

    class DynamicVH(val binding: DynamicListItemBinding) : RecyclerView.ViewHolder(binding.root)

    private class DynamicDiffCallback : DiffUtil.ItemCallback<Dynamic>() {
        override fun areItemsTheSame(oldItem: Dynamic, newItem: Dynamic): Boolean {
            return oldItem.id == newItem.id
        }

        override fun areContentsTheSame(oldItem: Dynamic, newItem: Dynamic): Boolean {
            return oldItem == newItem
        }
    }
}
<?xml version="1.0" encoding="utf-8"?>
<layout xmlns:android="http://schemas.android.com/apk/res/android">

    <data>

        <import type="android.view.View" />

        <variable
            name="dynamic"
            type="com.cloud.disk.bundle.dynamic.Dynamic" />
    </data>

    <FrameLayout
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:orientation="horizontal"
        android:padding="10dp">

        <TextView
            android:id="@+id/tv_content"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:layout_gravity="left"
            android:text="@{dynamic.content}" />

        <TextView
            android:id="@+id/tv_date"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:layout_gravity="right"
            android:text="@{dynamic.formatDate}" />
    </FrameLayout>

</layout>
```

详细的代码见[Github](https://github.com/junbin1011/CloudDisk/commit/0497fd86cf4efbfa4a195c92b65051817d2d4cae)

### 6.集成验收测试

1. dynamic bundle模块发布1.0.1版本

```text
implementation 'com.cloud.disk.bundle:dynamic:1.0.1'
```

1. 其他模块同步发布修改注入问题

```text
implementation 'com.cloud.disk.bundle:file:1.0.2'
implementation 'com.cloud.disk.bundle:user:1.0.1'
```

1. 执行守护测试

我们发现因为binding生成的文件有异常

```text
Field <com.cloud.dynamicbundle.databinding.DynamicListItemBinding.mDynamic> has type <com.cloud.disk.bundle.dynamic.Dynamic> in (DynamicListItemBinding.java:0)
Method <com.cloud.dynamicbundle.databinding.DynamicListItemBinding.getDynamic()> has return type <com.cloud.disk.bundle.dynamic.Dynamic> in (DynamicListItemBinding.java:0)
Method <com.cloud.dynamicbundle.databinding.DynamicListItemBinding.setDynamic(com.cloud.disk.bundle.dynamic.Dynamic)> has parameter of type <com.cloud.disk.bundle.dynamic.Dynamic> in (DynamicListItemBinding.java:0)
```

增加过滤规则

```text
.*com.cloud.*.databinding.*ItemBinding.*
```

1. 运行检查

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ee19f3bc3d874990b54d299e4efa1c5c~tplv-k3u1fbpfcp-zoom-1.image)

## 总结

本篇介绍了动态模块团队将动态件主页切换至Kotlin代码、重构为MVVM架构，并且补充了自动化测试。经过重构后，团队的开发效率和版本质量有了明显的提升。但本地数据库的管理依旧还是大量的sql 语句拼写，非常不利于扩展及维护，编写自动化测试也非常麻烦。

下一篇，移动应用遗留系统重构（15）- 数据库重构示例篇。我们将继续对数据库进行重构改造。

