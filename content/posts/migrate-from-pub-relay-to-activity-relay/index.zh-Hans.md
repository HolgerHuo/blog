---
title: '迁移Pub-Relay至Activity-Relay'
date: 2022-05-08T12:22:00+00:00
description: 本篇文章将介绍如何将Pub-Relay中继迁移至Activity-Relay程序。
summary: 本篇文章将介绍如何将Pub-Relay中继迁移至Activity-Relay程序。
author: holger
categories:
  - 技术分享
tags:
  - activity-pub
  - 中继
  - bash
  - linux
  - 运维
  - 迁移
  - mastodon
  - pub-relay
keywords:
  - activity-pub
  - 中继
  - bash
  - linux
  - 运维
  - 迁移
  - mastodon
  - pub-relay
feature: feature.png
featureAlt: 一张Go语言对比Crystal语言的插图。
images:
  - feature.png
slug: migrate-from-pub-relay-to-activity-relay
type: posts
showComments: true
isCJKLanguage: true
draft: false
showTableOfContents: true
author: holger
---

在联邦宇宙中，有一类中继程序作用是将不同实例间的消息同步，使得不同实例间的数据可以大量交换而不必关注全部对方实例账号。

[Pub-Relay](https://github.com/noellabo/pub-relay/)是由Mastodon开发者开发的中继程序，由Crystal Lang编写，在联邦宇宙初期被广泛使用。但它稳定性不高，经常出现bug导致无法处理消息转发，而且实现语言不太流行，很多开发者不能对其进行有效修改，致使其功能匮乏，甚至不能对中继列表进行管理。

[Activity-Relay](https://github.com/yukimochi/Activity-Relay)则是一款新问世的由Golang编写而成的中继转发实现。它稳定性相对Pub-Relay更高，且扩展性更强，自带管理Cli可以进行实例屏蔽/移除等操作，还可以变为私有中继手动批准加入，~~唯一的美中不足就是没有提供统计API~~，迁移到Activity-Relay可以提高公共中继的整体稳定性及可用性。

{{< alert >}}
Activity-Relay相较Pub-Relay性能开销有一定提升。

以[DragonRelay](https://relay.dragon-fly.club)为例，2C3G VPS CPU占用 1~2% → 10~15% (27订阅/2超大实例)
{{< /alert >}}

## 准备工作 

根据Activity-Relay的[官方Wiki](https://github.com/yukimochi/Activity-Relay/wiki/01.-Install)编译安装，配置Systemd及反向代理程序(请跳过生成RSA密钥一步)。

## 迁移 

需要迁移的内容有：

  * 中继签名私钥
  * 订阅列表

### 私钥 

私钥直接使用原中继私钥即可，请注意如果原先私钥权限不是`0600`请更改为`0600` 

`chmod 0600 /path/to/pem`

### 订阅列表 

由于两个程序实现均采用Redis作为持久数据库且key结构相似，我们可以直接使用一下脚本进行迁移。首先将原数据库复制到新的namespace以便发生未知错误时能够迅速回退:

```bash
# https://stackoverflow.com/a/26142152/15081893
source_host=localhost
source_port=6379
source_db=0
target_host=localhost
target_port=6379
target_db=1

redis-cli -h $source_host -p $source_port -n $source_db keys \*subscription\* | while read key; do # copy only subscription related keys
    echo "Copying $key"
    redis-cli --raw -h $source_host -p $source_port -n $source_db DUMP "$key" \
        | head -c -1 \
        | redis-cli -x -h $target_host -p $target_port -n $target_db RESTORE "$key" 0
done
```

随后执行以下命令更新数据库到Activity-Relay格式:

```bash
target_host=localhost
target_port=6379
target_db=1

redis-cli -h $target_host -p $target_port -n $target_db --scan --pattern relay:subscription:* | \
    while read key; do
        activity_id=$(redis-cli -h $target_host -p $target_port -n $target_db hget $key follow_id)
        actor_id=$(redis-cli -h $target_host -p $target_port -n $target_db hget $key follow_actor_id)
        redis-cli -h $target_host -p $target_port -n $target_db hmset $key activity_id $activity_id actor_id $actor_id
        redis-cli -h $target_host -p $target_port -n $target_db hdel $key follow_id follow_actor_id state
    done
```

重载你的反代程序，随后大功告成！

## 前端界面 

前端界面可以参考[https://github.com/dragonfly-club/dragon-relay](https://github.com/dragonfly-club/dragon-relay)配置。

