---
published: true
layout: post
title: Redis Clusters
---
## Redis Clusters

Recently I have been spending sometime figuring out various clustering techniques that can be employed in setting up my redis cluster environment in cloud (AWS). Firstly, AWS had elasticache which has redis option but elasticache doesn't support cross-region replication and in doesn't allow more than 5 read replicas (maybe it will in future releases of elasticache) - which was not meeting our requirements so we decided to stand our own redis cluster on ec2 intances. This gave us more flexibilty but yes - we had to take care of monitoring and some setups ourself which otherwise could have been taken care by elasticache. But thats Ok because redis configurations are not very complicated and for monitoring, We were planning to use influxdb/telegraf combination and not cloudwatch.

Anyhow, I did couple of POCs around pure redis-sentinel cluster and realized that we need some proxies like HAProxy, TWEM Proxy (nutcracker), OneCache etc to make maximum preformance, resilience and availabilty.

Few important things to note down in redis:
- No master-master replication.
- A slave can't have multiple masters.
- One redis cluster can't have more than one master [twemproxy supports multi-masters but technically creates different clusters for each shard].

**Redis-Sentinel**

Sentinel provides HA to redis cluster without human intervention. Sentinel cluster monitors the master and in case master node goes down, Sentinel does polling among themselves to choose a slave to be promoted as new master. There are different configuration and setup and you can read more about those at [redis's portal](http://redis.io/topics/sentinel). 

![redis-sentinel.png](https://raw.githubusercontent.com/shahzheeb/shahzheeb.github.io/master/_posts/redis-sentinel.png)

In this setup, There is no external proxy or process which can distinguish between master/slave nodes. So the client is directly connecting to the master at all times for write/read operations. The disadvange of this setup is there is:
- client is connecting to single node for read/write operations which could be a performance bottleneck as slaves are sitting idle while they could have been very well used for read operations.
- This solution is not scalable because there is just one write/master in the infrasturcture.
- There is no external process/proxy to route read/write traffic.
- Clients needs to have the knowledge of sentinel clusters to get the current master node to connect to.

To overcome these challanges, We can use twem, haproxy, onecache etc. I did a comparision prototype on HAProxy & TWEM proxy (nutcracker). Here are my learnings 

**HAProxy:**

