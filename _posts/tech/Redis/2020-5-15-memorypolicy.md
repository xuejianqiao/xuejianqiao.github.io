---
layout: post
title: Redis 内存淘汰策略
category: 中间件
tags: Redis,redis
keywords: 
description:
---

###一、 Redis 配置内存


#### 1、 通过 redis.conf 配置

```java
//设置Redis最大占用内存大小为100M

maxmemory

100mb

```


#### 2. 通过命令动态进行修改

```java
//设置Redis最大占用内存大小为100M

127.0.0.1:6379> config set maxmemory 100mb

//获取设置的Redis能使用的最大内存大小

127.0.0.1:6379> config get maxmemory

```



>  PS：如果不设置最大内存大小或者设置最大内存大小为0，在64位操作系统下不限制内存大小，在32位操作系统下最多使用3GB内存


### 二、Redis的内存淘汰

既然设置了Redis 最大占用内存的大小，那么配置的内存就有用完的时候。那在内存用完的时候，还继续往Redis里面添加数据不就没内存可用了吗？Redis定义了几种策略用来处理这种情况：

> * noeviction(默认策略) :对于写请求不再提供服务，直接返回错误。

> * allkeys-lru:从所有key中使用LRU算法进行淘汰。

> * volatile-lru：从设置了过期时间的key中使用LRU算法进行淘汰。

> * allkeys-random：从所有key中随机淘汰数据。

> * volatile-random：从设置了过期时间的key中随机淘汰。

> * volatile-ttl：在设置了过期时间的key中，根据key的过期时间进行淘汰，越早过期的越优先被淘汰。

#### 1、获取当前内存淘汰策略

```java

127.0.0.1:6379> config get maxmemory-policy


```

#### 2、通过配置文件设置淘汰策略 （修改redis.conf文件）

```java

maxmemory-policy allkeys-lru


```

#### 3、通过命令修改淘汰策略

```java

127.0.0.1:6379> config set maxmemory-policy allkeys-lru


```


### 三、 LRU 算法

> 核心思想：如果一个数据在最近一段时间没有被用到，那么将来被使用到的可能性也很小，所以就可以被淘汰掉。

#### 1.LRU在Redis中的实现


 **<font color="#ab2e2e">近似LRU算法：</font>** 
 
 Redis使用的是近似LRU算法，它跟常规的LRU算法还不太一样。近似LRU算法通过随机采样法淘汰数据，每次随机出5（默认）个key，从里面淘汰掉最近最少使用的key。

可以通过maxmemory-samples参数修改采样数量：例：maxmemory-samples 10 maxmenory-samples配置的越大，淘汰的结果越接近于严格的LRU算法

Redis为了实现近似LRU算法，给每个key增加了一个额外增加了一个24bit的字段，用来存储该key最后一次被访问的时间。


 **<font color="#ab2e2e">Redis3.0对近似LRU的优化 ：</font>** 


Redis3.0对近似LRU算法进行了一些优化。新算法会维护一个候选池（大小为16），池中的数据根据访问时间进行排序，第一次随机选取的key都会放入池中，随后每次随机选取的key只有在访问时间小于池中最小的时间才会放入池中，直到候选池被放满。当放满后，如果有新的key需要放入，则将池中最后访问时间最大（最近被访问）的移除。


![image.png-170.9kB][1]



### 四.LFU 算法

LFU算法是Redis4.0里面新加的一种淘汰策略（Least Frequently Used）。它的核心思想是根据key的最近被访问的频率进行淘汰，被访问多的则被留下。LFU算法能更好的表示一个key被访问的热度。 最近被访问的次数越多，热度越高。

LFU一共有两种策略：


>* volatile-lfu：在设置了过期时间的key中使用LFU算法淘汰key

>* allkeys-lfu：在所有的key中使用LFU算法淘汰数据


  [1]: http://static.zybuluo.com/qxjbeyond/dijh5uibwjj6thfm0dcn92ly/image.png