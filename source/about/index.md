---
layout: "about"
date: 2018-05-21 16:25:16
author:     "spirytusz"
---

# 📚关于我的博客

这个博客的文章都是我的个人原创，偶尔会研究一些好玩的东西，然后写写博客分享出来~

文章看心情同步到[掘金](https://juejin.cn/user/1785262614274302/posts)；😆

# 👨‍💻关于我

我是一名Android开发工程师，工作不忙（在虾皮的时候。在字节基本没时间😭）会研究些小玩具，发到github或者博客上；

初中的时候自己搞过反编译把离线壁纸App内的广告给ban掉（是广告先动的手，广告真是太流氓了😫），原理是反编译出smail代码，全局替换http链接为不合法链接；

大学毕业后，我尝试把Lua虚拟机集成到Android App上，实现在app能把Lua脚本跑起来的功能，写了篇博客：[记Android层执行Lua脚本的一次实践](/AndroidLua)介绍了下原理。最后写了个小玩具放在github上，仓库地址是[AndroidLua](https://github.com/spirytusz/AndroidLua)。Android App支持把Lua脚本的这个功能有很多有趣的地方，可以实现类似React-Native的效果，也可以做一些自动化脚本，有兴趣可以clone下来玩玩~

后来，我研究了gson对json的序列化和反序列化原理，给gson写了一个TypeAdapter编译器生成器，拿到了不小的技术收益。仓库地址是[GsonBooster](https://github.com/spirytusz/GsonBooster)：
* 生成前后的对比benchmark可以看仓库内的[README.md](https://github.com/spirytusz/GsonBooster/blob/master/README.md)；
* 由于生成代码较为复杂，我也编写了单元测试[ksp_test](https://github.com/spirytusz/GsonBooster/tree/master/booster-processor/processor-ksp/src/test/java/com/spirytusz/booster/processor/ksp/test)、[kapt_test](https://github.com/spirytusz/GsonBooster/tree/master/booster-processor/processor-kapt/src/test/java/com/spirytusz/booster/processor/kapt/test)验证是否按预期生成代码，测试用例可以戳一戳[这里](https://github.com/spirytusz/GsonBooster/tree/master/base/processor-base-test/src/main/resources)~

如果担心[GsonBooster](https://github.com/spirytusz/GsonBooster)会影响编译速度，也可以尝试用用IDE插件[GsonBooster-Plugin](https://github.com/spirytusz/GsonBooster-Plugin)。[GsonBooster-Plugin](https://github.com/spirytusz/GsonBooster-Plugin)和[GsonBooster](https://github.com/spirytusz/GsonBooster)的底层原理是相同的，唯一的区别是GsonBooster-Plugin的接入层是Intellij平台提供的[PSI](https://plugins.jetbrains.com/docs/intellij/psi.html)能力，而GsonBooster的接入层是gradle plugin；

工作的时候搞过插件化，也搞过跨平台的React Native和Flutter，最近的一份工作是搞直播的性能优化和架构优化，专注于进房的性能优化和直播插件与直播容器的解耦。

# 💼简历

目前我正在找工作，目标base地是**深圳**，目标岗位的Android开发工程师。我也对全栈开发有兴趣（我不会说：我的理想是我的代码能跑在更多设备上，服务更多人），如果能接受我转岗全栈开发，我也很乐意效劳~！😀

如果能帮我内推，不胜感激！😊

以下是我的中英文简历，敬请阅览~

* <a href="/res/resume.pdf">中文</a>
* <a href="/res/resume_en.pdf">English</a>


[//]: # (#### 公众号)

[//]: # ()
[//]: # (<img src="/img/Wechat_blog.jpeg" align="left"/>)
