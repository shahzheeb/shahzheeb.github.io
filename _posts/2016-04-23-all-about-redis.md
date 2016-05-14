---
published: true
layout: post
title: Redis Clusters
---
## Redis Clusters

Recently I have been spending sometime figuring out various clustering techniques that can be employed in setting up my redis cluster environment in cloud (AWS). Firstly, AWS had elasticache which has redis option but elasticache doesn't support cross-region replication and in doesn't allow more than 5 read replicas (maybe it will in future releases of elasticache) - which was not meeting our requirements so we decided to stand our own redis cluster on ec2 intances. This gave us more flexibilty but yes - we had to take care of monitoring and some setups ourself which otherwise could have been taken care by elasticache. But thats Ok because redis configurations are not very complicated and for monitoring, We 