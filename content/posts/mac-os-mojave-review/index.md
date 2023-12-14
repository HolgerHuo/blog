---
title: 'macOS Mojave 浅谈'
date: 2019-07-05T16:00:00+00:00
description: macOS Mojave 正式版已经发布了，有哪些全新内容以及更好的使用体验？脱离macOS生态依旧的Holger能否找回mac的手感？为什么新版macOS让他爱不释手？
summary: macOS Mojave 正式版已经发布了，有哪些全新内容以及更好的使用体验？脱离macOS生态依旧的Holger能否找回mac的手感？为什么新版macOS让他爱不释手？
categories:
  - 科技测评
tags:
  - apple
  - macos
  - mojave
keywords:
  - apple
  - macos
  - mojave 
thumbnail: feature.jpg
thumbnailAlt: 一张展示macOS版本的屏幕截图
images:
  - feature.jpg
slug: mac-os-mojave-review
expiryDate: 2019-07-05T16:00:00+00:00
type: posts
showComments: true
isCJKLanguage: true
draft: false
showTableOfContents: true
---

## 前言 

{{< figure
    src="system_info.jpg"
    alt="System Info"
    caption="System Info"
>}}

笔者手上有一台 2014 年中的 MacBook Pro 作为平时码字的工具，最初两三年一直使用的 macOS。从 Yosemite 时代一直到 Sierra，被 macOS 独树一帜的风格以及出色的稳定性所折服。但是由于种种原因，macOS 响应速度越来越缓慢、只有一台苹果设备所导致的生态链不完善、以及 Office 365 支持过于羸弱促使我由 macOS 切换到了盗版 Win 10。几天前，正好到了手机重置的时刻，心血来潮又刷回到了最新的 macOS Mojave。到现在刚好也用了一整天了，浅谈一下使用感受。

## 安装时的一点小插曲 

由于笔者之前安装 Win10 时抹掉了整个硬盘，在安装 macOS 的时候只能使用网络恢复。然而使用长城宽带重试了三四遍重新安装都无法下载组件，最后切换到联通宽带才得以解决。  
熬过了一个小时的安装过程，登陆 iCloud 又出现了问题。由于不知何时开启了 Apple ID 的双重认证，登陆时就一本正经的向我所谓的**受信任设备**发送了验证码，然而笔者已然说过，这台 MBP 是我唯一苹果生态下的产品，这个验证码当然收不到了。最后不得已只能新建了一个本地账户，想要先升级到 Mojave，然而在 App Store 中又需要输入验证码。最后和苹果客服打了半个多小时的电话才得以解决。有着强迫症的笔者认为多了一个本地账户的 macOS 是不**Purified**，所以又重新抹了一遍硬盘全新安装？。

## 依旧是那个 macOS 

随着那熟悉的**duang～**，久违的 macOS 又出现在了我的视线。Dock、 Finder 等应用悉数出现在我面前。一切都是那么熟悉，一切又是那么陌生。

## 使用初感 

### 终于支持 OneDrive 按需下载了！！！ 

{{< figure
    src="files_on_demand.jpg"
    alt="Files On-Demand"
    caption="Files On-Demand"
>}}

Windows 上早已推出的按需下载功能可以有效的节省磁盘空间，特别是对于使用 256G SSD 甚至是 128G 的 MBA 用户。在最新的正式版 macOS Mojave 中，OneDrive 终于支持了这一功能（理论上来讲，文件系统同是 APFS 的 High Sierra 也应该支持）。美中不足的是，Photos 等原生应用并不支持串流 OneDrive 中的视频。当然了，这毕竟是苹果的生态，Photos 支持的是 iCloud，就像 Windows 中支持 OneDrive 一样。

### 耳目一新的 App Store 

{{< figure
    src="app_store.jpg"
    alt="App Store"
    caption="App Store"
>}}

{{< figure
    src="editor_choice.jpg"
    alt="Editor's Choice"
    caption="Editor's Choice"
>}}

终于，App Store 进化了！相比于之前主页仅有几个少的可怜的滚动推荐，App Store 在首页加入了超大占比的 Banner，同时整合在左侧的侧边栏也更加符合时下审美。

类似于 Play Store 中 Editor’s Choice 的应用推荐可以帮助用户更加快捷的找到心仪的应用，只可惜 App Store 中的应用体量依旧不够庞大，希望在打通了 iOS 和 macOS 之后能够有所改观。

### macOS 的触控板 Windows 永远也赶不上 

在用惯了 Windows 10 的双指翻页之后，五指捏合、四指切换虚拟桌面等再熟悉不过的操作再一次清晰了起来。尽管 Windows 中最近加入了 Precision TouchBar 驱动，有许多功能还是只有 macOS 才能体验得到。双指放大缩小、全方位移动图片 PDF 让我彻底摆脱了鼠标的束缚。一般操作只需要键盘即可完工，只有当连上机械键盘码字时才有可能用到无线鼠标。

## 整体感受 

距离安装完 Mojave 已经过去两天了。这两天里，无论是用 Chrome 浏览浏览网页（真的要感谢 Chrome 插件所提供的跨平台支持，我在浏览习惯上并没有太大改变），还是用 MWeb 进行码字（MWeb 真心赞），抑或是用自带的 Terminal 配置服务器，macOS 都给我带来了超凡的体验。如下便是我使用 macOS 的一些感想。

### Pros 

#### Time Machine 备份服务 

{{< figure
    src="time_machine.jpg"
    alt="Time Machine"
    caption="Time Machine"
>}}

尽管当今的操作系统出现什么 Bug 的几率极其渺小，不只是什么强迫症所迫的我每一个操作系统都会使用备份服务（Windows 是 Dism + 还原点），macOS 所提供的 Time Machine 给我提供了完全无缝的备份体验。你随手用着电脑，连上一块外置硬盘（我所采用的是 sparseimage 备份到 Windows 的方案），顺便再插上电源，你的电脑就开始备份了。几分钟一次的增量同步丝毫不会占用过多的空间，到现在我的备份也不过 27G。

#### 内置的应用吊打一切第三方 

{{< figure
    src="terminal.png"
    alt="Terminal"
    caption="Terminal"
>}}

macOS 内置的各大应用都是可圈可点的，比如：Terminal。无论是作为 Unix 的 Terminal 进行系统部署，抑或是利用 ssh 连接远程桌面，terminal 都能带来完美的无缝体验。不需要第三方应用就可以原生支持 ssh 客户端以及 ftp 等服务的 Terminal 应该是各大系统的目标了（Windows Terminal 在我的台式机上怎么也打不开？）。

#### 耗电、稳定性・・・MBP 就得用 macOS 

{{< figure
    src="system_usage.png"
    alt="System Usage"
    caption="System Usage"
>}}

习惯了 Windows 一直插电使用用回 macOS 之后，几乎再也不用担心找不到电源线了。一天使用之后，第二天早餐前插上电源，吃过早餐就还你一个充满活力的 MBP，这恐怕也只有在 MBP 上的 macOS 才能做到了。  
此外，使用 macOS 之后散热器就被我扔到了角落。平时载着 Chrome 十几个网页再加上一个 VSCode 或者是 MWeb，后台听着 QQ 音乐，同步着 OneDrive，温度也不过四五十度。

#### macOS UI 整合度高 

Windows 下，各大应用软件的 UI 各不相同，甚至采用的安装器也不同（Install Wizard、自解压等等），这就导致增加了大量的学习成本。然而 macOS 确是高度整合，大部分的应用 UI 几乎都一致（当然，这也被一部分用户所诟病）。

### Cons 

#### 无法匹配的键盘 

由于系统原因，macOS 与 Windows 平台上的键盘并不一致，比如 Windows 上的 Alt 变为了 Command，这就涉及了快捷键的问题（Ctrl+C 变为了 Command+C）。使用过程中，经常会出现想要复制时按下了 Control+c 却实际上没有任何操作直到粘贴时才发现。  
更加鬼畜的是，连上机械键盘之后，很多键位都无法使用比如 Home，End。在敲代码时造成了一定的不便。这其中最难以接受的还是 Command 和 Alt 不能被识别为同一键位。这样在截图时无法使用机械键盘，只能将已经慵懒放松的双手从机械键盘移到笔记本键盘上进行截图。

#### 有些开源软件无法适配 

比如笔者在日常非常高频的一款图书管理软件 Calibre 在运行时经常时运行 10 分钟后崩溃掉（或许是软件自身原因，因为更新前并无此状况），这就导致 Calibre 抓取 RSS 的任务无法顺利完成，不过这也促使我寻找了在 Kindle 上更加合适的 RSS 阅读方式 Reabble（这是后话）。  
初步判定此类软件 bug 是由于跨平台使用同一代码不同编译器导致的（Calibre 基于 Python 开发），希望在下一版本中能够有所改观。

#### OneDrive 同步会出现莫名其妙的权限问题 

当笔者使用 OneDrive 同步相册、Calibre 书库，或者是 LastPass 的数据库时，Windows 上出现了极其令人蛋疼的无法删除现象。OneDrive 提示我：OneDrive Cannot Access to Your OneDrive Folder. Please Reconfigure it. 打开查看之后果然如此，我连打都打不开！右键属性发现所有人竟然是别看了，什么也没有，所有人上什么都没有！试遍了所有文件暴力删除工具都没有（Unlocker）。差点就以为要格式化硬盘才能解决，最后重启了电脑，总算解决了问题。

## 总结 

无论是 macOS、Windows 还是 Linux，每一款系统都有它的优点。不置可否的是，每一台设备，都会有它对应适宜的操作系统，这也就是为什么我会选择用回了 macOS。
