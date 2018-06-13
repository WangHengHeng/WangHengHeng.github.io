---
layout:     post
title:      "CODING 代码合并 - 企业版与个人版合并为一个项目"
date:       2018-06-12 10:48:01 +0800
author:     "王哼哼"
categories: 工作记录
---

经常使用 CODING 的用户应该会知道，CODING 在 App Store 有两个 App，分别对应 [个人版](https://itunes.apple.com/cn/app/id923676989) 和 [企业版](https://itunes.apple.com/cn/app/id1191398741)。当初做企业版的时候，基于当时的考虑，是在原项目的基础上分离出了一个项目来做企业版；而我现在需要做的，就是把企业版的代码，在合并到个人版去。

当时的考虑是：

- 个人版已经开源了，然而企业版并没有开源的打算
- 企业版这个产品线，我预期应该会有一些独有的功能，或者是一些与个人版有区分度的功能

但是现在为什么又要合到一起呢？很简单，我预估错了。错在哪里了呢：

- 在企业版被分离出去的这一年多时间里，初期是做了一些适配企业版的一些功能；但是后续的大部分时间里，基本上都是在同步着上线相同的功能
- 做的功能点相同，但是代码有分离在两处，这种感觉，拷贝粘贴得很酸爽
- 很重要的一点是，我发现，领导对于企业版的开源，并不抗拒！！！

这也说明了，在做某些决策的时候，是需要慎之又慎的。其实上面的错误，多想一下很容易就能想到：个人版和企业版只是面向的用户群不同而已，在业务逻辑上并没有什么差别（对于私有部署来说也是一样的）；在没有业务差别的情况下，设计上也没必要去做不同的设计，去割裂用户习惯；所以个人版和企业版 App 需要上线的功能也基本上是相同的。不过分离项目也有一个好处，就是个人版的开源项目 [Coding-iOS](https://github.com/Coding/Coding-iOS) 可以一直保持代码更新。

## 合并计划

两个分开了一年多的仓库，合并起来可不是个小事情，动手之前需要定个大概的计划先。我在 CODING 新建了一个私有 repo，然后把两个本地仓库分别都推到了这个 repo 的不同分支上，然后看一下差异，大概是下面这个样子：

![](/assets/2018/06/mr-diff.png)

差不多有 269 + 198，四百多个 commit 的差异，短时间内肯定弄不完，不过考虑到新版本刚刚提交上线，Android 又请假了一周，难得的空挡，可以搞一搞。不过，直接用 Pull Request 的方式去合并的话，只 project 就不是我能掌控的了，可能最后会合死去了。

我的想法是，对个人版和企业版的代码分别做一个拷贝，主要用来做比较（比较工具用 Xcode 的 FileMerge 就好）；对个人版的代码再做一个拷贝，在这个拷贝的基础上，手动合并与企业版之间的差异。拷贝好了之后，接下来的步骤就是配置 Target 和参数，然后再分模块合并代码了。

## 配置 Target 和参数

既然代码要合在一起，肯定要有一个在一起的方案；配置好了方案之后，才能把代码往一块放。我的方案就是用不同的 target 去分别对应到个人版和企业版。

### 新建 Target

打开工程文件，选中 target，右键 Duplicate 一个新的，默认名是 target+copy 样式的。可以选中，Enter 键重命名一下。 然后再新建一个对应的 Scheme，弄完之后大概是下面这样：

![](/assets/2018/06/new-target.png)

### 配置 Target

与配置有关的，主要是 info.plist 和预编译文件 .pch，把企业版里面对应的文件拷贝过来，重命名之后，在企业版 target 里面配置一下相应的路径。

> Build Setting 里面搜索键值 INFOPLIST_FILE、GCC_PREFIX_HEADER

还有一个需要配置的是 Universal Links（这个功能个人版是支持的，但是企业版不支持），对它的设置是在 Capabilities 的 Associated Domains 属性里。诡异的事情是我发现，在某个 target 下面勾了开关之后，会对另一个 target 有影响。后来发现对应 Capabilities 的配置文件是 .entitlements，像上面那样把企业版里对应的文件拷贝过来，设置一下配置路径就好。

> .entitlements 的设置对应键值是 CODE_SIGN_ENTITLEMENTS

### 添加宏

代码既然要放到一起了，那肯定少不了对 个人版/企业版 的判断，我选择根据 CFBundleIdentifier 去判断 target。宏定义像下面这样：

```
#define kTarget_Enterprise [[[NSBundle mainBundle] objectForInfoDictionaryKey:@"CFBundleIdentifier"] isEqualToString:@"net.coding.CodingEnterpriseForiOS"]
```

可是上面这个宏，之后运行时才能取到值，项目中必然还会存在需要基于条件编译的代码。我找了一下，但是没找到编译时判断自定义 target 的好办法（系统 TargetConditionals.h 文件里面倒是定义了不少判断系统 target 的宏）。所以我只在企业版对应的 .pch 文件中定义了宏：

```
#define Target_Enterprise
```

这样，在代码里就可以用类似下面的语句去执行条件编译：

```
#ifdef Target_Enterprise
//企业版代码

#else
//个人版代码

#endif
```


## 分模块合并

合并之前，我用 FileMerge 大概看了一眼，发现一个蛋疼的问题：当初我可能是因为有一点代码洁癖的原因，修改了企业版的项目名。嗯，到了要还的时候了，改文件头信息、文件夹名称...balabala

根目录里面，bootstrap、Podfile 是不着急改的，.gitignore 我们在上面一步已经改过了（主要是为了 ignore 掉 .pch 配置文件），其他的文件貌似都不用去管。代码主要集中在 Coding-iOS 文件夹，先把不用处理的文件标记一下，得到的结果如下图：

![](/assets/2018/06/file-merge-all.png)

> FileMerge 工具只是我用来标记代码合并的进度的，对于它的处理结果和结果存放位置，我都是不关心的

剩下需要处理的文件有：

```
- [ ] AppDelegate.m
- [ ] Controllers
- [ ] Ease_2FA
- [ ] Images
- [ ] Images.xcassets
- [ ] Launch Screen.xib
- [ ] Models
- [ ] Resources
- [ ] Util
- [ ] Vendor
- [ ] Views
```

按这个顺序去处理的话，肯定是不行的，因为代码耦合的原因，顺序处理不好的话，改着改着可能项目就一直跑步起来了。我的原则是，根据依赖关系：越被别个依赖，就越早处理；依赖的别个越多，就越晚去处理。然后，大概整理出下面这个处理顺序

```
//    工具类和第三方库：改动差异小，独立，被依赖的多
- [ ] Util
- [ ] Vendor

//    数据模型：简单，独立，被依赖的多
- [ ] Models

//    视图：依赖模型，又被控制器依赖
- [ ] Views

//    视图控制器：依赖模型和视图（其实 Ease_2FA 是相对独立了，但也属于是 Controller）
- [ ] Ease_2FA
- [ ] Controllers

//    AppDelegate 里面的东西太杂了，代码结构没写好，越写越大
- [ ] AppDelegate.m

//    启动页：独立，不过不影响运行。处理的话还要弄 target 之类的设置，放后面弄
- [ ] Launch Screen.xib

//    下面基本就是资源文件了，不影响编译运行，量多，放最后处理
- [ ] Resources
- [ ] Images.xcassets
- [ ] Images
```

下面就是一步步来了

### 工具类和第三方库

#### Util/Comon

这里面，大都是些 UI 改动，配色和适配 iPhone X 的。个人版这边的代码比较新，全部不用改动，Use left 标记一下已改（记录一下进度）：

![](/assets/2018/06/file-merge-util-common.png)

#### Util/Manager

标蓝的那几个，都是网络相关的，是依赖于 Model 的，需要稍后再处理；只把已处理的标记一下。

![](/assets/2018/06/file-merge-util-manager.png)

#### Util/OC_Category

基本上都是扩展，增量 merge 就好，个别需要处理一下的也很简单。

> 看到了当初小洁癖，添加的一些消除警告的代码；再看下目前项目里的警告数量。。唉！

只有 NSObject+Common 对 CodingNetAPIClient 依赖的略多了，先不管（其实应该把相应的方法挪到 Net 那边去的）

#### Vendor

第三方库，改的不少，不过改动都不大，还好。

### 数据模型

太多了，呃 166 项。

除了 Login 里面代码堆积的有点严重，其它都还好，只是有点耗时而已，毕竟，太多了。

弄完之后，先不忙着去处理视图。把上面没弄完的收尾一下先，主要是 NSObject+Common 和 CodingNetAPIClient 之间的相互依赖，还有依赖于 Model 的网络相关的几个 Manager

```
├── Coding_FileManager.h
├── Coding_FileManager.m
├── Coding_NetAPIManager.h
├── Coding_NetAPIManager.m
```


> 处理 Coding_NetAPIManager 的时候，我发现了一个棘手的东西：项目文件
> 
> 项目文件 这个模块，API 和 视图层，企业版 用的是新的，个人版 是旧的，要同步的话，需要一些工作量；我的本意是，这次合并应该只是简单的合并，尽量少做大规模的修改。但是在处理 Coding_NetAPIManager 的时候，我发现，当初企业版做的时候，偷懒沿用了个人版的方法声明，但是修改了参数类型（企业版里 folder 和 file 合并为同一个 file 类了）
> 这就是我遇到的问题，APIManager 不再是简单的添加 API 就能 merge 的了，而且还影响到了视图层的调用。纠结之后，决定：既然这些东西，都是 个人版 是迟早需要做的，那就在这次合并里做了好了，嗯！


整理完 Coding_NetAPIManager，感觉掉血很多。。o(╯□╰)o


### 视图

还是按照依赖关系来确定处理顺序：CCell - Cell - TableListView。这三个文件夹之外的文件，基本上都是挺独立的，处理顺序可以比较随意，我是放在最前面处理掉了。

总体整理下来还算顺利。最麻烦的是 Cell，主要是因为文件太多了。

### 视图控制器

```
- [ ] Ease_2FA
- [ ] Controllers
```

按部就班的合并就好，主要是量多，还有就是 对文件模块 的修改。

### AppDelegate

有一些涉及到配置的东西，个人版/企业版 需要分情况处理的情景还是挺多的，不过还好，代码量不算太大，处理起来很快。


### 资源文件

Launch Screen.xib：把企业版的拷贝过来，设置不同 target
Resources：主要是 html 的样式模板，以 Coding_iOS 的样式为准就好
Images.xcassets：除了 AppIcon 需要设置一下，其他的用 Coding_iOS 的就好（LaunchImage 可以删掉了）
Images：量多，不过好在很独立。有些同名图片，对应到 个人版/企业版 的真实文件又不同，我是新建了一个 images_diff 文件夹，用设置 target 的方式去管理了


## 总结

到这里，整个合并过程基本算是完成了，后续肯定还会有一些对于遗漏的 fix。涉及到的改动，大概是下面这样：

```
> [master 10e12431] 企业版代码合并至个人版 - 已合并除 images 之外的资源
> 204 files changed, 9853 insertions(+), 2358 deletions(-)
> [master 435ba96c] 企业版代码合并到个人版 - 图片资源整理
>  121 files changed, 658 insertions(+), 268 deletions(-)
```

整个合并过程历时很久，不过我觉得还是值得的，至少以后新的代码改动，都可以免去在两个项目之间的复制粘贴了。在合并过程中，我也发现了一些设计不合理的地方，但是对于怎么改，我还是不太确定，只能慢慢去整合了。

这篇博客，就算做是个开始吧...
