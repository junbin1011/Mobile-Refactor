# 开篇

## 前言

2008年9月22日，谷歌正式对外发布第一款Android手机。苹果公司最早于2007年1月9日的MacWorld大会上公布IOS系统。移动应用领域的发展已经超过10年。在[App Annie 最新的移动市场报告](https://www.appannie.com/cn/insights/market-data/mobile-2021-new-records-beckon/)中分享2020应用下载量已经达到**2180亿次**，同比增加7%。根据[Statista](https://www.statista.com/statistics/289418/number-of-available-apps-in-the-google-play-store-quarter/#:~:text=Google%20Play%3A%20number%20of%20available%20apps%20as%20of%20Q4%202020&text=This%20statistic%20gives%20information%20on,compared%20to%20the%20previous%20quarter.)的统计，2020年度Google Play的应用数量为**3148932**个。

在移动互联网的高速发展及竞争中，更快及更高质量的交付用户，显然尤为重要。但很多产品随着移动互联网的发展，已经迭代超过10年。在这个过程中人员流动、技术债务累计、技术生态更新，使得产生了大量的遗留系统。就像一辆低排量的破旧汽车，再大的马路，技术再好的驾驶员，达到车辆本身的系统瓶颈，速度就很难再提升起来。

## 遗留系统

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d71ae09f397e470181ee920425eacb1e~tplv-k3u1fbpfcp-zoom-1.image)

在以往的项目中，遇到了大量的这种遗留系统。这些系统具有以下一些特点。

* 大泥球架构，代码量上百万行，开发人员超过100+
* 内部耦合高，代码修改维护牵一发动全身，质量低
* 编译集成调试慢，没有任何自动化测试，开发效率低
* 技术栈陈旧，祖传代码，无人敢动

在这样的背景下，个别少的团队选择重写，当然没有良好的过程管理及方法，好多重写完又成了新的遗留系统。也有的团队选择重构，但目前相关的方法及教程比较少。这里推荐一下[《重构（第2版）》](https://book.douban.com/subject/33400354/)，书中有基本的重构手法。另外一本[《修改代码的艺术》](https://book.douban.com/subject/2248759/)，书中有很多基于遗留代码开发的示例。但对于开发人员来说，缺少比较贴近移动应用项目实战的重构指导及系统方法。很多团队依旧没有解决遗留系统根本的原因，仅靠不断的堆人，恶性循环。

## CloudDisk 演示示例

CloudDisk是一个类似于Google Drive的云存储应用。该应用主要拥有3大核心业务模块，文件、动态及个人中心。

该项目已经维护超过10年以上，目前有用开发人员100+。目前代码在一个大模块中，约30w行左右，编译时间10分钟以上。团队目前主要还面临几个问题。

1. 开发效率低

> 编译时间长，经常出现代码合并冲突。遗留大量技术债务，团队疲于交付需求

1. 代码质量差

> 经常修改出新问题，版本提测问题多，没有任何自动化测试

1. 版本发布周期长

> 往往需要1个月以上，市场反馈响应慢

我们希望通过一个更贴近实际工程项目的浓缩版遗留系统示例，持续解决团队在产品不断迭代中遇到的问题。从架构设计与分析、安全重构、基础生态设施、流水线、编译构建等方面，一步一步介绍如何进行持续演化。我们将通过文章及视频演示的方式进行分享，希望通过这个系列文章，大家可以更系统的掌握移动应用项目中实战的重构技巧及落地方法。

## 大纲

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/6f82487e860346448589ed670b97319b~tplv-k3u1fbpfcp-zoom-1.image)

## 关于

**欢迎关注CAC敏捷教练公众号**。微信搜索：**CAC敏捷教练**。

* 作者：黄俊彬 
* [博客：junbin.tech](https://junbin.tech/)
* [GitHub: junbin1011 ](https://github.com/junbin1011)
* [知乎: @JunBin](https://www.zhihu.com/people/junbin-9-77)

