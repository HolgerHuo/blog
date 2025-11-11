---
title: 'How-to: Migrate from Pub-Relay to Activity-Relay'
date: 2022-05-09T12:18:23+00:00
description: This guide instructs you to migrate from Pub-Relay to Activity-Relay.
summary: This guide instructs you to migrate from Pub-Relay to Activity-Relay.
author: holger
categories:
  - Programming
feature: feature.png
featureAlt: 
images:
  - feature.png
slug: migrate-from-pub-relay-to-activity-relay
type: posts
showComments: true
isCJKLanguage: false
draft: false
showTableOfContents: true
---
In the fediverse, there are some nodes functioning as bridges across the fediverse so that data among different instances are synchronized without having to follow all accounts in an instance.

[Pub-Relay](https://github.com/noellabo/pub-relay/" rel="noreferrer noopener) was developed by Mastodon developers and was created in Crystal Lang, which had been generally adopted in the early stage of Fediverse. But the program itself often throws errors in production causing statuses to freeze and the implementation language is rarely used, resulting in the lack of customization and extension, even the function of managing subscription lists.

[Activity-Relay](https://github.com/yukimochi/Activity-Relay), on the other hand, is a new, Golang implementation of public relay. It has achieved better stability and is more extensible. The built-in CLI can block, remove, etc instances from subscription, as well as turn the public relay into a private one for personal use. ~~The only drawback is that it lacks the stats API for frontend communication.~~ Anyway, migrating to Activity-Relay will result in higher stability and availability for your public relay.

{{< alert >}}
  Activity-Relay requires slightly more resource consumption than Pub-Relay.<br />Take for example, the CPU usage rose from 1~2% to 10~15% on [DragonRelay](https://relay.dragon-fly.club) (27 Subscribers/ 2 Giant Instances)
{{< /alert >}}

## Preparation 

Build binary, and set up your Activity-Relay environment according to [Official Documentation](https://github.com/yukimochi/Activity-Relay/wiki/01.-Install). (Please skip generating RSA private key)

## Migration 

All you need to migrate are:

  * the private key for signatures
  * Your subscribers

### Private Key 

Simply use your original key used by pub-relay. Do keep in mind that if the key was not in `0600` permission please chmod it

`chmod 0600 /path/to/pem`

### Subscribers 

We may use the following script to migrate DB directly as both softwares utilize Redis as persistent storage and have a similar key structure. First, you need to copy the original database into a new namespace in case you encounter any issues:

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

Then execute these commands to perform a database migration:

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

Reload your reverse proxy and that it is!

## Front-end 

You may refer to [https://github.com/dragonfly-club/dragon-relay](https://github.com/dragonfly-club/dragon-relay) for a frontend example.

