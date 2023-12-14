---
title: '从我的世界服务器到Google Public DNS —— 记一次艰难的Debug'
date: 2023-08-13T17:19:38+00:00
description: 本篇文章记录了博主从Debug我的世界服务器连接性问题到发现Google Public DNS故障的过程。
summary: 本篇文章记录了博主从Debug我的世界服务器连接性问题到发现Google Public DNS故障的过程。
categories:
  - 技术分享
tags:
  - clash
  - debug
  - dns
  - google
  - ipv6
  - minecraft
  - udp
keywords:
  - clash
  - debug
  - dns
  - google
  - ipv6
  - minecraft
  - udp
thumbnail: thumbnail.jpg
thumbnailAlt: 一张展现了AI回应人类错误的信息的截图。
images:
  - thumbnail.jpg
slug: from-minecraft-to-google-public-dns
type: posts
showComments: true
isCJKLanguage: true
draft: false
author: holger
showTableOfContents: true
---

## tl;dr 

[~~Google Public DNS~~](https://developers.google.com/speed/public-dns) [Google DNS64](https://developers.google.com/speed/public-dns/docs/dns64) 位于IPv6协议(`2001:4860:4860::6464`、`2001:4860:4860::64`)的服务器在解析AAAA记录时，如果目标域名没有AAAA记录存在，会~~随机返回一个内部IPv6地址~~返回一个6to4 IPv6地址，如:

```PowerShell
PS > nslookup -q=aaaa holger.net.cn 2001:4860:4860::6464
Server:  dns64.dns.google
Address:  2001:4860:4860::6464

Non-authoritative answer:
Name:    holger.net.cn
Address:  64:ff9b::2ac1:6c96
```

```bash
$ curl -H "accept: application/dns-json" "https://[2001:4860:4860::64]/resolve?name=holger.net.cn&type=aaaa"
{"Status":0,"TC":false,"RD":true,"RA":true,"AD":true,"CD":false,"Question":[{"name":"holger.net.cn.","type":28}],"Answer":[{"name":"holger.net.cn.","type":28,"TTL":300,"data":"64:ff9b::2ac1:6c96"}]}
```

(也可直接访问 [https://dig.ping.pe/holger.net.cn:AAAA:2001:4860:4860::6464](https://dig.ping.pe/holger.net.cn:AAAA:2001:4860:4860::6464))

而~~正常的DNS响应~~非6to4转换的DNS响应(如Google Public DNS位于IPv4协议上的服务器)则为:

```bash
$ curl -H "accept: application/dns-json" "https://8.8.8.8/resolve?name=holger.net.cn&type=aaaa"
{"Status":0,"TC":false,"RD":true,"RA":true,"AD":true,"CD":false,"Question":[{"name":"holger.net.cn.","type":28}],"Authority":[{"name":"holger.net.cn.","type":6,"TTL":477,"data":"kristin.ns.cloudflare.com. dns.cloudflare.com. 2317199504 10000 2400 604800 1800"}]}
```

```PowerShell
PS > nslookup -q=aaaa holger.net.cn 8.8.4.4
Server:  dns.google
Address:  8.8.4.4

Name:    holger.net.cn

```

(也可直接访问 [https://dig.ping.pe/holger.net.cn:AAAA](https://dig.ping.pe/holger.net.cn:AAAA) )

~~错误的~~6to4DNS响应如果要被复现，须满足以下条件:

  1. 请求的服务器必须为Google Public DNS IPv6服务器: `2001:4860:4860::6464`、`2001:4860:4860::64`，请求方式不限(已测试：Plain DNS/DoH)；
  2. 目标请求主机不存在AAAA记录；
  3. 请求时指定请求AAAA记录。

注： 本篇文章部分信息出现了乌龙事件，具体乌龙分析请看文章末端及[评论](#comment-47)

## 前情提要 

几天前升级了[DragonCraft](https://minecraft.dragon-fly.club/join)我的世界服务器并配置了[https://github.com/GeyserMC](Geyser)插件以实现在Minecraft PE中连接至服务器游玩。在本地测试中，很奇怪的发现在移动端经常会无法连接至服务器，具体原因不详。由于是MC PE端新出现的问题，我围绕MC PE传输协议进行了一系列Debug，最终发现问题的根源实际在于Google Public DNS。

## 网络环境介绍 

博主家中的网络环境IPv4网关由Clash TProxy以fake-ip透明代理完全接管，IPv6则直连上游网关，本地网络中关闭IPv6 DNS服务器，DNS请求交由Clash进行并开启IPv6开关。

理想状态下，所有域名相关的请求都将由Clash fake-ip接管，并由Clash自行决定连接IPv6/IPv4；IPv4直连也将由Clash网关，v6则直接交由上游处理。

之所以没作为IPv6网关是由于笔者使用的DHCP服务器无法根据客户端MAC地址分配不同的IPv6网关及DNS服务器，故无法无感为某些设备设置为直连流量，故放弃代理全部v6流量。

## 开始Debug 

首先，博主试验了不同网络环境下MC PE服务器的连接情况，并把出现此bug的条件锁定到家中的TProxy环境，无论是开启移动数据/固定宽带并启动Tun模式Clash，或是直连移动数据/固定宽带，此问题均无法复现。

查看Clash日志，发现到MC服务器的流量始终只有上传没有下载，即没有从服务器返回的数据(其实此处应该尝试抓取一下Minecraft PE内部的日志数据，但在这一步懒了orz)。于是又登入Minecraft服务器后台，发现确实没有来自我所在IP地址的访问记录。

由于从其他网络环境及其他设备的测试正常，故基本排除掉MC服务端的故障。

在此部分Debug中，博主发现了主界面时有大概率无法登入XBox Live账户(此处埋下伏笔)，于是把它列为了另一项并列的待解决Bug。

从仅有上传没有下载流量且服务端没有收到来自客户端的访问记录，博主猜测可能是UDP链路链接出现了某些问题，随后测试其他UDP链接，发现并无过多异常(WeChat Video Call)，但Windows的NTP时间自动同步模块失效了，并在连接记录中发现也是同样的仅上传无下载流量。于是沿着这条思路继续测试，发现time.windows.com连向了远端代理。经过简单调查，发现是由于远端代理UDP不支持(Clash.Meta 代理链问题)导致。然而我的世界服务器连接并未流向远端代理，故重新回到起点。

由于`time.windows.com`处在fake-ip-filter列表中(即ClashDNS不返回FakeIP地址而返回真实地址)，笔者联想到是否由于Minecraft PE检测到返回地址为内部保留地址便无法连接(实际上联想的的也有点偏)，于是编辑Minecraft中填写的MC服务器地址为ip，发现Bug依旧，故继续整理思路。

鉴于平时对UDP流量测试较少，笔者决定从UDP展开调查，首先检查了用于透明代理的iptables规则，发现其可以正常工作(显而易见)，然后对尝试了一些其他UDP端口及服务器的连通性，发现并没有什么线索，Debug过程在此陷入瓶颈。

## 转入正确思路 

由于UDP方向暂时无可用思路，笔者决定先去Debug先前发现的XBox无法登录问题。首先博主检阅了在冷启动MCPE至没有成功登录过程中，MCPE发起过的请求。请求大多数连接至`a.b.xboxlive.com`一类的域名，笔者推断这个域名主要和登录相关。由于XBox有部分域名在fake-ip-filter列表中，笔者觉得可能是因为fake-ip-filter列表缺少部分登陆相关域名导致客户端丢弃了返回的本地地址，造成无法登录。但随后的调查发现所有XBox Live相关域名均返回的是未经处理的ip地址，此思路也放弃。

继续，笔者推断或许是由于某些https连接没能被正确启动，于是笔者开始在本地电脑上测试出现过的域名，直到

```
PS > ping title.mgt.xboxlive.com

Pinging title.mgt.xboxlive.com [64:ff9b::2ac1:6c96] with 32 bytes of data:
Request timed out.
Request timed out.
Request timed out.
Request timed out.

Ping statistics for 64:ff9b::2ac1:6c96:
    Packets: Sent = 4, Received = 0, Lost = 4 (100% loss)
```

看到这里笔者大吃一惊，为什么Ping到了一个内部IPv6地址？难道Clash Fake-IP开始有IPv6支持了？？怪不得链接不到，本地的IPv6网络并没有被网关到Clash啊！！！冷静下来思考一下，明明这个域名在fake-ip-filter列表中啊，就算是IPv4的结果返回的也是真实IP，v6就更不可能返回Fake-IP了啊。

笔者在此处定论(定的太早了orz): 这应当是Clash内核本身的某种Bug。

在翻看了各Clash内核的Issues、Changelogs后，笔者完全没有发现可能存在的相关联内容。笔者甚至拉取了Clash内核源码，对DNS模块进行研读，但也没有找到更新的思路。于是再次向前推：难道会不是Clash的问题吗？但这个DNS结果是从哪里来的？？？

博主将Clash内核日志模式提升至DEBUG档，在仔细浏览后发现，这条记录竟然是来自于真实DNS解析！

```
[DNS] title.mgt.xboxlive.com --> [64:ff9b::2ac1:6c96], from https://[2001:4860:4860::6464]:443/dns-query
```

我的天！！！这是什么DNS！为什么只有他给出了结果，还是一个并不正确的结果！！！

## 结果验证 

经过搜索，笔者发现此IPv6地址来自于~~Google Public DNS~~ Google DNS64。这让笔者更加难以相信眼前的结果，于是笔者在本地电脑和远程服务器上分别进行了本文开头的验证，发现属实能够复现，到此基本确定，~~问题应当出自Google Public DNS~~问题应当出于错误的选用了DNS64服务器。

## 结论梳理 

在从Clash上游DNS列表中移除~~Google Public DNS IPv6版本~~Google DNS64的服务器后，XBox终于可以正确登录了。可喜可贺的是，在成功登录XBox Live账号后，Minecraft服务器也能够实现一次成功登录，终于解决了困扰笔者很久的两个问题。

本次Debug过程比较曲折，一是由于某些地方思路因为懒惰没有直接调取内部日志，其次由于不了解Minecraft服务器登录过程，导致忽略了真实造成Minecraft服务器登陆失效的原因，以及在某些思路上出现判断失误。

~~此Issue已被反馈至[Google Issue Tracker](https://issuetracker.google.com/issues/295695504)。~~

## 跟进 

在评论区有网友指出作者先前所给出的IPv6 DNS服务器实为Google DNS64 服务器，经过调研，笔者发现原先所用的DNS服务器确实为Google DNS64而非Google Public DNS。本篇文章的疏忽大致是由以下几点原因造成：

  1. 作者在定位到DNS服务器返回IP地址时，仅意识到其为保留地址而忽略了其为6to4地址；
  2. 作者在进一步查询DNS服务器相关信息时，错误的相信了由AI生成的答案
{{< figure
  src="ai_false_answer.jpg"
  alt="一张展现了AI回应人类错误的信息的截图"
  style="zoom: 80%"
  caption="AI回应人类错误的信息"
>}}
  3. 作者在得出结论时，忽略了反向验证这一关键步骤，即没有验证Google Public DNS IPv6的真实地址。


从本次事故作者深刻地意识到AI生成技术的不靠谱性orz，在此呼吁广大网友在相信AI结果之前务必反向验证其准确性。

