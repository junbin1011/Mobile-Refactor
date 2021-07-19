# 制品管理篇

## 前言

上一篇我们提到CloudDisk团队决定将团队划分为5个团队，分别管理文件、动态、用户中心、平台及公共库。整个项目不再统一使用一个Git大仓进行代码管理，每个团队能独立维护自己的代码仓库。

但目前虽然各个模块已经解耦完成，但还是以源码的依赖的形式进行编译。我们要将业务拆分独立的仓库，需要把源码的依赖都调整为二进制的依赖才行。本篇我们就对CloudDisk进行改造，解开源码的编译依赖。

## 二进制发布配置

这里我们选择常用的maven进行管理。为了方便演示，我们使用本地maven进行配置。实际项目中我们只要把地址设置为服务器的maven地址则可，在模块的build.gradle添加代码如下：

```java
apply plugin: 'maven'

group = 'com.cloud.disk'
archivesBaseName = 'library'
version = '1.0.0'

repositories.mavenCentral()

uploadArchives {
    repositories.mavenDeployer {
        repository(url: 'file:'+ rootDir.getPath()+'/lib')
    }
}
```

在根目录的build.gradle添加代码如下：

```java
allprojects {
    repositories {
        maven{
            url 'file:'+ rootDir.getPath()+'/lib'
        }
    }
}
```

工程中就可以引用本地maven发布的jar包了。

触发版本发布,执行如下命令:

```text
 ./gradlew :library:uploadArchives
```

## CloudDisk重构示例

具体Library改造示例代码见[Github](https://github.com/junbin1011/CloudDisk/commit/0d3a6a51bee60c479a43e013259ff2e08b1d04cb)

这里我们看看改造后的App Gradle文件dependency如下：

```java
implementation 'com.cloud.disk:library:1.0.0'
implementation 'com.cloud.disk:api:1.0.0'
implementation 'com.cloud.disk:platform:1.0.0'

implementation 'com.cloud.disk.bundle:file:1.0.0'
implementation 'com.cloud.disk.bundle:dynamic:1.0.0'
implementation 'com.cloud.disk.bundle:user:1.0.0'
```

## 总结

基于前面我们做了大量的解耦工作，本篇我们通过本地maven进行二进制管理，实际项目中这些aar都在远程的maven中管理。我们也可以将各个业务特性在独立的仓库中管理。

在架构设计中App是整体工程的集成，用于集成及最后的发布。分仓后所有的业务Bundle在独立的仓库中维护，我们希望Bundle能够进行独立的编译调试，下一篇移动应用遗留系统重构（12）- 编译调试篇，我们将继续对CloudDisk进行改造，让业务Bundle支持独立的编译调试。

