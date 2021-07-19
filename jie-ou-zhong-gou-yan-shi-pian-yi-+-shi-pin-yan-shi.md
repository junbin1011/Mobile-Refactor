# 解耦重构演示篇\(一\)+视频演示

## 前言

[移动应用遗留系统重构（5）- 重构方法篇](https://juejin.cn/post/6952298178095874055)中我们分享了重构流程，主要为4个操作步骤。 1. 识别一个内聚的包 2. 解除该包的异常依赖 3. 移动该包对应的代码及资源到新的模块 4. 包解耦验收

[移动应用遗留系统重构（6）- 测试篇](https://juejin.cn/post/6954635678982340622)中我们为[CloudDisk](https://github.com/junbin1011/CloudDisk)补充了一组基本的冒烟测试。有了基本的测试守护后，本篇我们将挑选library（基础组件库）及file（文件业务模块）2个包进行重构演示。文章中包含操作的流程，同时了录制了视频。

**library模块重构代码演示视频：**[https://mp.weixin.qq.com/s/C0nQRbgmp1cLhwoWzGpqFw](https://mp.weixin.qq.com/s/C0nQRbgmp1cLhwoWzGpqFw)

**file模块重构代码演示视频：**[https://mp.weixin.qq.com/s/xbAIu6bWS7pLLAgHe-a8Bg](https://mp.weixin.qq.com/s/xbAIu6bWS7pLLAgHe-a8Bg)

## 安全重构演示

### library包重构

1. 依赖分析

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/4e5378783e684dbe9ef579d82727e3cf~tplv-k3u1fbpfcp-zoom-1.image)

通过分析我们发现library包存在对上层模块的反向依赖，该依赖为异常依赖，须要解除。

1. 安全重构

重构前代码：

```text
public class HttpUtils {
    public static void post(String url) {
        //发送http post请求，需要用到userId做标识
        String params = UserController.userId;
    }
    public static void get(String url) {
        //发送http get请求，需要用到userId做标识
        String params = UserController.userId;
    }
}
```

重构手法：**提取方法参数**

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/7fd063ccc36240af9dc917dc22347067~tplv-k3u1fbpfcp-zoom-1.image)

重构后代码：

```text
public class HttpUtils {
    public static void post(String url, String userId) {
        //发送http post请求，需要用到userId做标识
        String params = userId;
    }
    public static void get(String url, String userId) {
        //发送http get请求，需要用到userId做标识
        String params = userId;
    }
}
```

1. 代码移动

代码移动至独立的library，加上对应的Gradle依赖：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/7710d9f223be476aa59e3aff799797b8~tplv-k3u1fbpfcp-zoom-1.image)

移动后模块结构如下：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/0a956bec3eae4972856462f21e8b81be~tplv-k3u1fbpfcp-zoom-1.image)

1. 功能验证

执行冒烟测试，验证功能

```text
./gradlew app:testDebug --tests SmokeTesting
```

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b6e9797c72ae489d9f3770540214c98b~tplv-k3u1fbpfcp-zoom-1.image)

**代码演示：**[https://mp.weixin.qq.com/s/C0nQRbgmp1cLhwoWzGpqFw](https://mp.weixin.qq.com/s/C0nQRbgmp1cLhwoWzGpqFw)

具体的代码：[github链接](https://github.com/junbin1011/CloudDisk/commit/852463774f69142cb2ecb2b20cc581da6e1f3db7)

### file包重构

1. 依赖分析

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b4d79404f0744a138079ec6ab4feb43c~tplv-k3u1fbpfcp-zoom-1.image)

通过分析我们发现file包存在横向bundle模块的依赖，该依赖为异常依赖，须要解除。

1. 安全重构 重构前代码：

   ```text
   public class FileController {
    public List<FileInfo> getFileList() {
        return new ArrayList<>();
    }

    public FileInfo upload(String path) {
        //上传文件
        LogUtils.log("upload file");
        HttpUtils.post("http://file", UserController.userId);
        return new FileInfo();
    }

    public FileInfo download(String url) {
        //下载文件
        if (!UserController.isLogin) {
            return null;
        }
        return new FileInfo();
    }
   }
   ```

   重构手法：

2.1 抽取getUserId、isLogin方法

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/3127d85339284f74a542d401c3d9b094~tplv-k3u1fbpfcp-zoom-1.image)

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/887a5f718256483d88bdceeefba8e1f4~tplv-k3u1fbpfcp-zoom-1.image)

重构后代码如下：

```text
public class FileController {
    public List<FileInfo> getFileList() {
        return new ArrayList<>();
    }

    public FileInfo upload(String path) {
        //上传文件
        LogUtils.log("upload file");
        HttpUtils.post("http://file", getUserId());
        return new FileInfo();
    }

    public FileInfo download(String url) {
        //下载文件
        if (!isLogin()) {
            return null;
        }
        return new FileInfo();
    }

    private String getUserId() {
        return UserController.userId;
    }

    private boolean isLogin() {
        return UserController.isLogin;
    }
}
```

2.2 抽取代理类，UserState

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/3e72483a1bf3490b8d75bf712311053d~tplv-k3u1fbpfcp-zoom-1.image)

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/dcac912c9b2f42c199366538e423b180~tplv-k3u1fbpfcp-zoom-1.image)

重构后代码如下：

```text
public class FileController {
    private final UserState userState = new UserState();

    public List<FileInfo> getFileList() {
        return new ArrayList<>();
    }

    public FileInfo upload(String path) {
        //上传文件
        LogUtils.log("upload file");
        HttpUtils.post("http://file", userState.getUserId());
        return new FileInfo();
    }

    public FileInfo download(String url) {
        //下载文件
        if (!userState.isLogin()) {
            return null;
        }
        return new FileInfo();
    }
}
```

2.3 抽取接口

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/8e84f1de3a1449b8900c1b4abe095da4~tplv-k3u1fbpfcp-zoom-1.image)

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/71d6b21294224bb48b72b4cef27e01c8~tplv-k3u1fbpfcp-zoom-1.image)

重构后代码如下：

```text
public class FileController {
    private final UserState userState = new UserStateImpl();

    public List<FileInfo> getFileList() {
        return new ArrayList<>();
    }

    public FileInfo upload(String path) {
        //上传文件
        LogUtils.log("upload file");
        HttpUtils.post("http://file", userState.getUserId());
        return new FileInfo();
    }

    public FileInfo download(String url) {
        //下载文件
        if (!userState.isLogin()) {
            return null;
        }
        return new FileInfo();
    }
}
```

2.4 提取构造函数，依赖接口注入

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f17a4ab2aab04fab8c944ef9d73a7865~tplv-k3u1fbpfcp-zoom-1.image)

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a2ba19a9cb9e411bb688b055d3f786e2~tplv-k3u1fbpfcp-zoom-1.image)

重构后代码如下：

```text
public class FileController {
    private final UserState userState;

    public FileController(UserState userState) {
        this.userState = userState;
    }

    public List<FileInfo> getFileList() {
        return new ArrayList<>();
    }

    public FileInfo upload(String path) {
        //上传文件
        LogUtils.log("upload file");
        HttpUtils.post("http://file", userState.getUserId());
        return new FileInfo();
    }

    public FileInfo download(String url) {
        //下载文件
        if (!userState.isLogin()) {
            return null;
        }
        return new FileInfo();
    }
}
```

同样FileFragment我也也做相同的构造及接口注入。

1. 代码移动

同样使用moduraize进行移动，代码移动至独立的fileBundle，加上对应的Gradle依赖： ![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/689a6b71b4e047bc874394b57de33dc5~tplv-k3u1fbpfcp-zoom-1.image) 移动后模块结构如下：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/23ff0cf8c77843b4bb43de4a124ce5d5~tplv-k3u1fbpfcp-zoom-1.image)

1. 功能验证

   执行冒烟测试，验证功能

```text
./gradlew app:testDebug --tests SmokeTesting
```

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d85f4d6b12a84b1b9eabf0394e1d52ac~tplv-k3u1fbpfcp-zoom-1.image)

> 每一小步重构时都可以频繁运行测试，提前发现问题

**代码演示：**[https://mp.weixin.qq.com/s/xbAIu6bWS7pLLAgHe-a8Bg](https://mp.weixin.qq.com/s/xbAIu6bWS7pLLAgHe-a8Bg)

具体的代码：[github链接](https://github.com/junbin1011/CloudDisk/commit/b69bbc8e7d5d83b0c542ada06d7157a6fb997682)

## 总结

本篇我们按着重构的4个步骤，借助IDE的安全重构，小步的重构了library和file2个包。虽然依赖解除了，代码也移动到独立的模块里面。但是我们还是发现了一些问题。

1. UserState接口的层层注入，我们需要手工维护了好多构造方法及对应的注入
2. App依赖了fileBundle的Fragment，UI跳转上存在编译的依赖

下一篇，单体移动应用“模块化”演进之旅（8）- 依赖注入篇，我们将分享常见的注入方式及业内优秀的实践，并对DiskCloud继续进行改造优化。

