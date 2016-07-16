---
published: true
layout: post
title: Redis Clusters
---
## Redis Clusters

Recently I have been spending some time figuring out various clustering techniques that can be employed in setting up my redis cluster environment in cloud (AWS). Firstly, AWS had elasticache which has redis option but elasticache doesn't support cross-region replication and in doesn't allow more than 5 read replicas (maybe it will in future releases of elasticache) - which was not meeting our requirements so we decided to stand our own redis cluster on ec2 intances. This gave us more flexibilty but yes - we have to take care of monitoring and some setups ourself which otherwise could have been taken care by elasticache. But thats Ok because redis configurations are not very complicated and for monitoring, We were planning to use influxdb/telegraf combination and not cloudwatch.

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

HAProxy is a matured and time tested proxy for number of platforms including Redis cache. HAProxy also has a web interface to monitor frontend/backend traffic. In my setting, I created two HAProxies - One for write (which has master) and other one for read only. This allows to horizontally grow the cluster for reads and directing all write traffic to write cluster. The nodes on the read HAproxy cluster are basically slaves of the master node. When master goes down, the write proxy will automatically choose the new master (without the need of any external process though the new master is chosen using Redis's sentinel process).

Proxy's backend configuration for **WRITE**:

This will select only the node which is master to forward the traffic. "tcp-check expect string role:master" will make sure that this backend connects to the node which is currently a master.

```json
frontend ft_redis
	bind *:6378 name redis
	default_backend bk_redis

backend bk_redis
	balance first
	option tcp-check
	tcp-check connect
	tcp-check send PING\r\n
	tcp-check expect string +PONG
	tcp-check send info\ replication\r\n
	tcp-check expect string role:master
	tcp-check send QUIT\r\n
	tcp-check expect string +OK
	server redis_6380 localhost:6380 check inter 1s 
	server redis_6381 localhost:6381 check inter 1s
	server redis_6388 localhost:6388 check inter 1s 
	server redis_6389 localhost:6389 check inter 1s
```

Proxy's backend configuration for **READ ONLY**:
 
```json
frontend ft_redis
	bind *:6379 name redis
	default_backend bk_redis

backend bk_redis
	option tcp-check
	tcp-check connect
	tcp-check send PING\r\n
	tcp-check expect string +PONG
	tcp-check send info\ replication\r\n
	tcp-check expect string role:slave
	tcp-check send QUIT\r\n
	tcp-check expect string +OK
	server redis_6388 localhost:6388 check inter 1s 
	server redis_6389 localhost:6389 check inter 1s
	server redis_6380 localhost:6380 check inter 1s 
	server redis_6381 localhost:6381 check inter 1s
```
 
In write cluster, At any time, the traffic will be directed to any ONE of the node (master) while readonly cluster will have n-1 nodes listening at all times.

The advantage of using HAProxy is that we don't need any external process to moitor the switching-over of master and re-configure the HAProxy's settings to point to new master.

The limitations on the other hand is that if your requirement is of very high volume writes i.e. greater than 100K WPS then it is difficult to scale becasue we can't have more than one write cluster. In those cases, You may have to consider using twem proxy by twitter.

**Twem Proxy:**

Twitter's official version of [twem proxy (nutcracker)](https://github.com/twitter/twemproxy) doesn't support Sentinel and also the community is not very active. 

[Sentinel not support by official twem](https://github.com/twitter/twemproxy/issues/297)

Though I found a forked repository of twem which works with Sentinel and the maintainers are very active.

[Git link to TwemProxy with Sentinel](https://github.com/ifwe/twemproxy)
