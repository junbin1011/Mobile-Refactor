# 路由篇

## 前言

上一篇[移动应用遗留系统重构（8）- 依赖注入篇 ](https://juejin.cn/post/6963214120178941983)最后我们通过IDE的依赖分析发现，App模块主界面直接依赖了file Bundle的FileFragment，存在直接的编译依赖。

跨模块间Activity或Fragment的直接依赖是最常见的。但是如果有直接的依赖，我们就无法做到业务模块独立编译调试，后续做动态化也没办法统一管理。本篇我们主要分为3个部分，第一部分是路由的原理，第二部分是业内优秀的路由框架实践，最后我们将继续对CloudDisk中UI跳转进行重构。

## 路由原理

Android中常用的页面跳转就是通过直接的依赖方式。

```text
Intent intent = new Intent(MainActivity.this, LoginActivity.class);
startActivity(intent);
```

但其实Intent还有另外一个API支持使用类名进行隐式跳转。

```text
Intent intent = new Intent();
intent.setClassName(this,"com.cloud.disk.platform.login.LoginActivity");
startActivity(intent);
```

这种方式就不会存在编译的问题。但当整个应用内的页面跳转量很大时，我们就很难全局进行统一维护。并且很多场景需要动态推送页面跳转，我们需要统一管理所有页面的地址，这个时候我们就需要有统一的方案进行路由管理。

那么如何进行统一的管理呢？其实一个很自然的思路就是建立一个统一的映射，例如：

```text
uri://user/login -> com.cloud.disk.platform.login.LoginActivity
```

然后通过一个统一的方式进行管理，这就是所谓的路由表。当应用进行跳转时，输入虚拟的地址，经过路由表进行查询得到实际的地址，然后就可进行跳转。并且有了这一层转换，我们就可以做很多扩展，例如降级、拦截等等

下面就让我们一起来看看一些业内的优秀实践。

## 业内优秀实践

### [ARouter](https://github.com/alibaba/ARouter)

ARouter主要采用的也是路由表的方式，具体的使用和原理，网上有很多资料。这里主要列出官网上介绍的一些主要的功能。

* 支持直接解析标准URL进行跳转，并自动注入参数到目标页面中
* 支持多模块工程使用
* 支持添加多个拦截器，自定义拦截顺序
* 支持依赖注入，可单独作为依赖注入框架使用
* 映射关系按组分类、多级管理，按需初始化

  支持用户指定全局降级与局部降级策略

  页面、拦截器、服务等组件均自动注册到框架

  支持多种方式配置转场动画

* 支持获取Fragment
* 完全支持Kotlin以及混编
* 支持第三方 App 加固\(使用 arouter-register 实现自动注册\)
* 支持生成路由文档
* 提供 IDE 插件便捷的关联路径和目标类

更多详细的介绍和使用说明，可以参考[Github上的介绍](https://github.com/alibaba/ARouter/blob/master/README_CN.md)

> 这里我们从Github上的介绍发现，同样采用了注解和Gradle插件在编译时生成文件，但ARouter并没有像Hilt那样有完善的测试套件支持，所以如果使用Robolectric在JVM上进行测试会有影响。

### [DeepLinkDispatch](https://github.com/airbnb/DeepLinkDispatch/)

DeepLinkDispatch是airbnb开源的一个路由框架，原理也是采用路由表的方式。

提供声明性的、基于注释的API来定义应用程序深度链接。 可以注册一个Activity来处理特定的深度链接，方法是使用@DeepLink和URI对其进行注释。DeepLinkDispatch将解析URI并将深度链接与URI中指定的任何参数一起发送到适当的Activity。

相比之下，功能没有ARouter强大，且国内的社区活跃度没有ARouter高，具体的使用方式可以参考[官方的介绍](https://github.com/airbnb/DeepLinkDispatch)

## CloudDisk路由重构示例

经过对比，我们决定使用功能相对强大且社区活跃度高的ARouter，对CloudDisk进行改造。具体的完整代码示例[Github](https://github.com/junbin1011/CloudDisk/commit/ab0bbc0785faf41817683e0b509e9c1b122ce7db)。这里我们贴出前后代码使用的比较。

改造前：

```text
fragments.add(FileFragment.newInstance());
```

改造后：

```text
//声明
@Route(path = "/bundle/file")
public class FileFragment extends Fragment 

//调用
fragments.add((Fragment) ARouter.getInstance().build("/bundle/file").navigation()）;
```

但当我们运行冒烟测试的时候发现出现空异常，如下

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/fa2e766a28f146ec80b5f54a862dbe95~tplv-k3u1fbpfcp-zoom-1.image)

ARouter的navigation并不能找到实例，上面我们有提到ARouter同样采用了注解和Gradle插件在编译时生成文件，但ARouter并没有像Hilt那样有完善的测试套件支持，在JVM上进行测试会有影响。这里我们采用的方案是Shadow，将实际ARouter的跳转Mock掉。

```text
@Implements(Postcard.class)
public class ShadowPostCard {

    @RealObject
    public Postcard postcard;

    @Implementation
    public Object navigation() {
        if ("/bundle/file".equals(postcard.getPath())) {
            try {
                return Class.forName("com.cloud.disk.bundle.file.FileFragment").newInstance();
            } catch (ClassNotFoundException | IllegalAccessException | InstantiationException e) {
                e.printStackTrace();
            }
        }
        return null;
    }
}
```

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/db5e42fda661495f8c6ccfca9975db57~tplv-k3u1fbpfcp-zoom-1.image)

> 我们还可以考虑把测试用例放入androidTest，使用真机运行测试。但为了得到更快的反馈速度，我们决定先沿用shadow的方案。

## 总结

使用路由除了能解耦开编译时的依赖，统一了路由地址也能更好的满足应用的跳转场景。目前CloudDisk已经解耦了lib和file bundle 2个模块，并且基础的注入和路由也已经有了，下一篇单体移动应用“模块化”演进之旅（10）- 解耦重构演示篇（二）我们将继续分享对platform、user、dynamic进行依赖解除重构，将会分享更多的实战解耦手法。

