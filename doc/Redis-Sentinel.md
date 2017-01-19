# redis + sentinel 实现主从集群搭建及容灾部署

[TOC levels=2,3]: # "Table of Contents"

### 目录
- [Redis简介](#introduce-to-Redis )
    - [什么是Redis](#introduce-to-redis)
    - [安装&配置](#features)
    - [常用命令](#features)
    - [主从复制](#two-tier-model)
    - [图形化客户端](#two-tier-model)
- [Sentinel简介](#release-road-map)
    - [什么是sentinel](#version-230)
    - [sentinel配置](#version-220)
    - [Version 2.1.1](#version-211)
    - [Version 2.1.0](#version-210)
    - [Source Update is Long Overdue](#source-update-is-long-overdue)
- [在Spring中使用集群](#older-versions)
    - [添加jedis支持包](#version-184)
    - [在spring中的配置](#rogues-gallery-of-features)
- [参考资料](#background)


## Redis简介

![Screenshot](https://redis.io/images/redis-white.png)



### Introduce to Redis

关于 **Redis**的简介还是直接上官网简介吧,英文比较原汁原味。

Redis is an open source (BSD licensed), in-memory data structure store,
used as a database, cache and message broker.
It supports data structures such as strings, hashes, lists, sets,
sorted sets with range queries, bitmaps, hyperloglogs and geospatial
indexes with radius queries. Redis has built-in replication,
Lua scripting, LRU eviction, transactions and different levels of
on-disk persistence, and provides high availability via Redis
Sentinel and automatic partitioning with Redis Cluster.  

:zzz: to be continued...


