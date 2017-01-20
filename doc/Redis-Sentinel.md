# redis + sentinel 实现主从集群搭建及容灾部署

[TOC levels=2,3]: # "Table of Contents"

### 目录
- [Redis简介](#introduce-to-Redis )
    - [什么是Redis](#introduce-to-redis)
    - [安装&配置](#install-and-config)
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

### Install and Config

Redis官方没有支持windows版,但是windows开源技术团队开发并维护了win64版,
想要研究Windows版可以参考[此处][2]。所以一下安装适合linux版或Mac。
此时最新稳定版是3.2.6  
打开终端输入以下命令下载最新版Reids
```bash
wget http://download.redis.io/releases/redis-3.2.6.tar.gz
```
解压
```bash
tar -xvf redis-3.2.6.tar.gz
```
进入解压后的redis文件夹
```bash
cd redis-3.2.6
```
编译
```bash
make
```
安装
```bash
make install
```
安装完成之后在命令行输入`redis`再连续两次tab键,将出现相应提示。
此时输入./redis-server。如下图

![image](https://github.com/johnxue2013/docs/blob/master/images/redis-make-install.png)


redis默认监听6379端口,关于为什么是这个端口号,是有故事的[传送门][3]。

:zzz: to be continued...



[1]:https://redis.io/download "redis下载地址"
[2]:https://github.com/MSOpenTech/redis "windows版redis GITHUB地址"
[3]:http://oldblog.antirez.com/post/redis-as-LRU-cache.html "redis 6379故事"
