# 数据库重构示例篇

## 前言

上一篇[移动应用遗留系统重构（14）- Kotlin+MVVM重构示例篇](https://juejin.cn/post/6977702335879315493)我们介绍了动态模块团队将动态件主页切换至Kotlin代码、重构为MVVM架构，并且补充了自动化测试。经过重构后，团队的开发效率和版本质量有了明显的提升。但本地数据库的管理依旧还是大量的sql 语句拼写，非常不利于扩展及维护，编写自动化测试也非常麻烦。

Google官方的建议是[我们强烈建议您使用 Room而不是 SQLite](https://developer.android.com/training/data-storage/room#db-migration-testing)。框架带来的好处是节省编写大量的模版代码，易于维护，同时面向接口，便于扩展。关于性能方面，有的同学可能认为，使用原生的SQL语句可以将性能优化到极致，但往往忽略了维护大量的SQL拼写语句也带来了非常大的成本，况且框架也支持自定义SQL语句。

接下来给大家分享，动态模块的本地缓存如何从Sqlite安全、渐进式重构至Room。

## 重构

```text
graph TD
A(1.梳理业务逻辑)-->B(2.分析原有的代码设计)
B-->C(3.补充守护测试)
C-->D(4.简单设计)
D-->E(5.小步安全重构)
E-->F(6.集成验收测试)
```

### 1.梳理业务逻辑

原有的缓存逻辑比较简单，主要提供2个功能。

* 查询缓存的动态数据
* 保存缓存的动态数据（保存上一次加载的所有数据）

### 2.分析原有的代码设计

```java
class LocalDataSource @Inject constructor(
        @ApplicationContext private var mContext: Context) : DataSource {

    //判断游标是否为空
    override fun getDynamicListFromCache(): List<Dynamic> {
        val dynamicList: MutableList<Dynamic> = ArrayList()
        val dataBaseHelper = DataBaseHelper(mContext)
        val c = dataBaseHelper.writableDatabase.query(DataBaseHelper.dynamic_info, null, null, null, null, null, null)
        if (c.moveToFirst()) { //判断游标是否为空
            for (i in 0 until c.count) {
                c.move(i) //移动到指定记录
                val id = c.getInt(c.getColumnIndex(DataBaseHelper.id))
                val content = c.getString(c.getColumnIndex(DataBaseHelper.content))
                val date = c.getLong(c.getColumnIndex(DataBaseHelper.date))
                dynamicList.add(Dynamic(id, content, date))
            }
        }
        return dynamicList
    }

    override fun saveDynamicToCache(dynamicList: List<Dynamic>?) {
        val dataBaseHelper = DataBaseHelper(mContext)
        dynamicList?.let {
            dataBaseHelper.writableDatabase.delete(DataBaseHelper.dynamic_info, null, null)
            for ((id, content, date) in dynamicList) {
                val cv = ContentValues()
                cv.put(DataBaseHelper.id, id)
                cv.put(DataBaseHelper.content, content)
                cv.put(DataBaseHelper.date, date)
                dataBaseHelper.writableDatabase.insert(DataBaseHelper.dynamic_info, null, cv)
            }
        }
    }
}
```

主要存在问题：

* 较多模版代码，不利于维护扩展
* 没有采用事务管理，极端大数据情况下，可能有异常
* 没有任何的守护测试

### 3.补充守护测试

```java
@RunWith(AndroidJUnit4::class)
@MediumTest
class LocalDataSourceTest {

    @Test
    fun `should get dynamic is empty when database has not data`() {
        //given
        val localDataSource = LocalDataSource(ApplicationProvider.getApplicationContext())
        //when
        val dynamicListFromCache = localDataSource.getDynamicListFromCache()
        //then
        assertThat(dynamicListFromCache).isEmpty()
    }

    @Test
    fun `should get dynamic success when database has data`() {
        //given
        val localDataSource = LocalDataSource(ApplicationProvider.getApplicationContext())
        localDataSource.saveDynamicToCache(getMockData())
        //when
        val dynamicListFromCache = localDataSource.getDynamicListFromCache()
        //then
        val dynamicOne = dynamicListFromCache[0]
        assertThat(dynamicOne.id).isEqualTo(1)
        assertThat(dynamicOne.content).isEqualTo("今天天气真不错！")
        assertThat(dynamicOne.date).isEqualTo(1615963675000L)
        val dynamicTwo = dynamicListFromCache[1]
        assertThat(dynamicTwo.id).isEqualTo(2)
        assertThat(dynamicTwo.content).isEqualTo("这个连续剧值得追！")
        assertThat(dynamicTwo.date).isEqualTo(1615963688000L)
    }


    private fun getMockData(): ArrayList<Dynamic> {
        val dynamicList = ArrayList<Dynamic>()
        dynamicList.add(Dynamic(1, "今天天气真不错！", 1615963675000L))
        dynamicList.add(Dynamic(2, "这个连续剧值得追！", 1615963688000L))
        return dynamicList
    }
}
```

### 4.简单设计

分步进行重构，小步验证。

1. 将SQLiteOpenHelper管理数据库替换成Room的管理，但还是维持原有的SQL操作方式
2. 将原有的SQL操作方式调整为ROOM的Dao形式
3. 使用Coroutine管理数据库的异步操作
4. 调整已有的测试用例

### 5.小步安全重构

#### 将SQLiteOpenHelper管理数据库替换成Room的管理，但还是维持原有的SQL操作方式

1. bean改造，注意字段与表名需要与原来一致

```java
@Entity(tableName="dynamic_info")
data class Dynamic(@PrimaryKey @ColumnInfo(name = "id") val id: Int,
                   @ColumnInfo(name = "content") val content: String,
                   @ColumnInfo(name = "date") val date: Long) {
    @Ignore
    val formatDate = DateUtil.getDateToString(date)
}
```

1. 使用SupportSQLiteOpenHelper进行管理，继续以writableDatabase进行管理

```java
class LocalDataSource @Inject constructor(
        @ApplicationContext private var mContext: Context) : DataSource {

    val db = Room.databaseBuilder(
            mContext,
            AppDatabase::class.java, "dynamic.db"
    ).build()


    //判断游标是否为空
    override fun getDynamicListFromCache(): List<Dynamic> {
        val dynamicList: MutableList<Dynamic> = ArrayList()
        val dataBaseHelper = db.openHelper
        val c = dataBaseHelper.writableDatabase.query("")
        if (c.moveToFirst()) { //判断游标是否为空
            for (i in 0 until c.count) {
                c.move(i) //移动到指定记录
                val id = c.getInt(c.getColumnIndex(DataBaseHelper.id))
                val content = c.getString(c.getColumnIndex(DataBaseHelper.content))
                val date = c.getLong(c.getColumnIndex(DataBaseHelper.date))
                dynamicList.add(Dynamic(id, content, date))
            }
        }
        return dynamicList
    }

    override fun saveDynamicToCache(dynamicList: List<Dynamic>?) {
        val dataBaseHelper = db.openHelper
        dynamicList?.let {
            dataBaseHelper.writableDatabase.delete(DataBaseHelper.dynamic_info, null, null)
            for ((id, content, date) in dynamicList) {
                val cv = ContentValues()
                cv.put(DataBaseHelper.id, id)
                cv.put(DataBaseHelper.content, content)
                cv.put(DataBaseHelper.date, date)
                dataBaseHelper.writableDatabase.insert(DataBaseHelper.dynamic_info, SQLiteDatabase.CONFLICT_REPLACE, cv)
            }
        }
    }
}
```

#### 将原有的SQL操作方式调整为ROOM的Dao形式

```java
class LocalDataSource @Inject constructor(
        @ApplicationContext private var mContext: Context) : DataSource {

    private val db = Room.databaseBuilder(
            mContext,
            AppDatabase::class.java, "dynamic.db"
    ).build()


    //判断游标是否为空
    override  fun getDynamicListFromCache(): List<Dynamic> {
        return db.dynamicDao().getAll()
    }

    override  fun saveDynamicToCache(dynamicList: List<Dynamic>?) {

        dynamicList?.let {
            db.dynamicDao().deleteAll()
            db.dynamicDao().insertAll(*it.toTypedArray())
        }
    }
```

#### 使用Coroutine管理数据库的异步操作

```java
@Dao
interface DynamicDao {
    @Query("SELECT * FROM dynamic_info")
    suspend fun getAll(): List<Dynamic>

    @Insert
    suspend fun insertAll(vararg dynamic: Dynamic)


    @Query("DELETE FROM dynamic_info")
    suspend fun deleteAll()
}
```

#### 调整已有的测试用例

```java
@ExperimentalCoroutinesApi
@RunWith(AndroidJUnit4::class)
@MediumTest
class LocalDataSourceTest {
    private val testDispatcher = TestCoroutineDispatcher()

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
    fun `should get dynamic is empty when database has not data`() = runBlocking {
        //given
        val localDataSource = LocalDataSource(ApplicationProvider.getApplicationContext())
        //when
        val dynamicListFromCache = localDataSource.getDynamicListFromCache()
        //then
        assertThat(dynamicListFromCache).isEmpty()
    }

    @Test
    fun `should get dynamic success when database has data`() = runBlocking {
        //given
        val localDataSource = LocalDataSource(ApplicationProvider.getApplicationContext())
        localDataSource.saveDynamicToCache(getMockData())
        //when
        val dynamicListFromCache = localDataSource.getDynamicListFromCache()
        //then
        val dynamicOne = dynamicListFromCache[0]
        assertThat(dynamicOne.id).isEqualTo(1)
        assertThat(dynamicOne.content).isEqualTo("今天天气真不错！")
        assertThat(dynamicOne.date).isEqualTo(1615963675000L)
        val dynamicTwo = dynamicListFromCache[1]
        assertThat(dynamicTwo.id).isEqualTo(2)
        assertThat(dynamicTwo.content).isEqualTo("这个连续剧值得追！")
        assertThat(dynamicTwo.date).isEqualTo(1615963688000L)
    }


    private fun getMockData(): ArrayList<Dynamic> {
        val dynamicList = ArrayList<Dynamic>()
        dynamicList.add(Dynamic(1, "今天天气真不错！", 1615963675000L))
        dynamicList.add(Dynamic(2, "这个连续剧值得追！", 1615963688000L))
        return dynamicList
    }
}
```

### 6.集成验收测试

1. 允行Dynamic模块所有守护测试，成功

   ```text
   ./gradlew dynamicBundle:testDUT
   ./gradlew dynamicDebug:testDUT
   ```

2. 编译运行DynamicDebug出现异常如下：

```text
 java.lang.IllegalStateException: Pre-packaged database has an invalid schema: dynamic_info(com.cloud.disk.bundle.dynamic.Dynamic).
     Expected:
    TableInfo{name='dynamic_info', columns={date=Column{name='date', type='INTEGER', affinity='3', notNull=true, primaryKeyPosition=0, defaultValue='null'}, content=Column{name='content', type='TEXT', affinity='2', notNull=true, primaryKeyPosition=0, defaultValue='null'}, id=Column{name='id', type='INTEGER', affinity='3', notNull=true, primaryKeyPosition=1, defaultValue='null'}}, foreignKeys=[], indices=[]}
     Found:
    TableInfo{name='dynamic_info', columns={date=Column{name='date', type='LONG', affinity='1', notNull=false, primaryKeyPosition=0, defaultValue='null'}, id=Column{name='id', type='INTEGER', affinity='3', notNull=false, primaryKeyPosition=1, defaultValue='null'}, content=Column{name='content', type='VARCHAR(1024)', affinity='2', notNull=false, primaryKeyPosition=0, defaultValue='null'}}, foreignKeys=[], indices=[]}
        at androidx.room.RoomOpenHelper.checkIdentity(RoomOpenHelper.java:163)
```

这是因为我们在测试阶段运行在JVM的环境上，没有提前发现数据迁移的问题。这里我们需要使用ROOM的Migration机制进行数据备份迁移。

```java
  private val MIGRATION_1_2 = object : Migration(1, 2) {
        override fun migrate(database: SupportSQLiteDatabase) {
            database.execSQL("ALTER TABLE dynamic_info RENAME TO dynamic_info_back_up")
            database.execSQL("CREATE TABLE dynamic_info ( id  INTEGER PRIMARY KEY NOT NULL, content TEXT NOT NULL,date INTEGER NOT NULL)")
            database.execSQL("INSERT INTO dynamic_info (id, content,date) SELECT id, content,date FROM dynamic_info_back_up")
        }
    }

    private val db = Room.databaseBuilder(
            mContext,
            AppDatabase::class.java, "dynamic.db"
    ).addMigrations(MIGRATION_1_2).build()
```

更多迁移及测试见[迁移 Room 数据库](https://developer.android.google.cn/training/data-storage/room/migrating-db-versions)、[从SQlite迁移到Room](https://developer.android.google.cn/training/data-storage/room/sqlite-room-migration)

## 总结

本篇我们分享了动态模块数据库从Sqlite迁移至Room的过程，使用框架节省编写大量的模版代码，易于维护及扩展。

前面CloudDisk团队已经拆分成了多仓库开发，但各个模块各自维护了第三方库。除了版本不统一、也导致打包构建时会带入多个版本的三方库。

下一篇，移动应用遗留系统重构（16）- Gradle依赖管理篇。我们将继续对CloudDisk的Gradle版本进行统一管理演示。

