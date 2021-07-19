# Gradle依赖管理篇

## 前言

上一篇[移动应用遗留系统重构（15）- 数据库重构示例篇](https://juejin.cn/post/6979198588630859806)我们提到CloudDisk团队已经拆分成了多仓库开发，但各个模块各自维护了第三方库。这样除了版本不统一、也导致打包构建时会带入多个版本的三方库。

由于目前各业务模块已经分仓开发，我们无法使用ext.的声明，统一版本号及定义库别名。我们只能以Gradle插件的形式发布到Maven库中，这样所有其他模块就可以直接apply使用。

## CloudDisk Gradle 插件改造

### Gradle 插件

1. 创建buildSrc模块
2. 创建BasePlugin类及LibPluginExtension
3. 创建resource/META-INF/gradle-plugin/com.cloud.disk.plugin.properties,声明class

   ```java
   implementation-class=com.cloud.disk.plugin.BasePlugin
   ```

   4.发布插件，在使用的工程apply

   ```java
   apply plugin: 'com.cloud.disk.plugin'
   ```

BasePlugin类如下：

```java
public class BasePlugin implements Plugin<Project> {

    public static Project mProject

    @Override
    void apply(Project project) {
        mProject=project
        project.extensions.create("Lib", LibPluginExtension.class)
    }
}
```

LibPluginExtension类如下,部分代码省略：

```java
public class LibPluginExtension {
    String roomVersion = '2.2.6'
    String lifeCycleVersion = '2.3.0'
    String hiltVersion = '2.31.2-alpha'
    //... ...

    void test() {
        BasePlugin.mProject.dependencies {
            testImplementation "junit:junit:$junitVersion"
            testImplementation "org.mockito:mockito-inline:$mockitoVersion"
            testImplementation "android.arch.core:core-testing:1.1.1"
            testImplementation "com.google.truth:truth:1.0.1"
            testImplementation 'com.tngtech.archunit:archunit-junit4:0.15.0'
            testImplementation "org.robolectric:robolectric:$robolectricVersion"
            testImplementation 'androidx.test:core:1.3.0'
            testImplementation 'androidx.test.ext:junit:1.1.2'
            testImplementation "androidx.test.espresso:espresso-core:$espressoVersion"
            testImplementation "androidx.test.espresso:espresso-intents:$espressoVersion"
            debugImplementation "androidx.fragment:fragment-testing:1.3.1"
        }
    }

    void room() {
        BasePlugin.mProject.dependencies {
            implementation "androidx.room:room-runtime:$roomVersion"
            implementation "androidx.room:room-ktx:$roomVersion"
            kapt "androidx.room:room-compiler:$roomVersion"
            testImplementation "androidx.room:room-testing:$roomVersion"
        }
    }

    void coroutines() {
        BasePlugin.mProject.dependencies {
            implementation "org.jetbrains.kotlinx:kotlinx-coroutines-core:$coroutinesVersion"
            implementation "org.jetbrains.kotlinx:kotlinx-coroutines-android:$coroutinesVersion"
            testImplementation "org.jetbrains.kotlinx:kotlinx-coroutines-test:1.3.3"
        }
    }


    // ... ...


}
```

### CloudDisk 改造

1. dynamicBundle模块修改如下：

   ```java
   dependencies {
    implementation 'com.cloud.disk:api:1.0.0'
    implementation 'com.cloud.disk:library:1.0.0'
    implementation 'com.cloud.disk:platform:1.0.0'

    project.Lib.kotlin()
    project.Lib.supportV4()
    project.Lib.appCompact()
    project.Lib.material()
    project.Lib.router()
    project.Lib.mvvm()
    project.Lib.ktx()
    project.Lib.coroutines()
    project.Lib.room()
    project.Lib.hilt()
    project.Lib.test()
   }
   ```

2. dynamicDebug模块修改如下：

```text
dependencies { implementation 'com.cloud.disk:api:1.0.0' 
implementation project(':dynamicBundle')
project.Lib.hilt() project.Lib.test() 
project.Lib.appCompact() 
project.Lib.material() 
project.Lib.constraintlayout()
project.Lib.kotlin() project.Lib.ktx() }
```

完整修改见[Github](https://github.com/junbin1011/CloudDisk/commit/8e1b7a8c228ecad2b6a1b14deba824cfe9cd80e4)

## 总结

本篇主要分享了 CloudDisk Gradle 插件改造，包含统一的三方库的版本管理及如何简化依赖调用，有效的解决了三方库版本不统一问题。

随着重构工作的持续，各个模块已经补充了足量的自动化测试，但CloudDisk团队目前的CI只提供打包的功能，供测试团队进行验证。分仓以后也没有对独立模块仓库建立流水线，测试团队还是只能在最后做集成的验收。

下一篇，移动应用遗留系统重构（17）- 流水线设计篇。我们将重新对CloudDisk进行流水线的设计，包含质量门禁检查及根据单个模块、APP集成，制定对应的策略。

