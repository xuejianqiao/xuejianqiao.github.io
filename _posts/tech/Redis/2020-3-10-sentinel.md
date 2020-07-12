---
layout: post
title: Redis哨兵模式
category: 数据库
tags: Redis,redis
keywords: 
description:
---

### 一、Redis的高可用概述

> 在 Redis 中 ，实现 **高可用** 的技术主要包含 <font color="#ab2e2e">**持久化、复制、哨兵、 集群**</font>   下面简单说明它们的作用，以及解决了什么样的问题：

 1. **<font color="#ab2e2e">持久化：</font>** 持久化是 **最简单的** 高可用方法。它的主要作用是 **数据备份**，即将数据存储在 **硬盘**，保证数据不会因进程退出而丢失。
 
 2. **<font color="#ab2e2e">复制：</font>** 复制是高可用 Redis 的基础，哨兵 和 集群 都是在 **复制基础上** 实现高可用的。复制实现了**数据的多机备份**以及对于读操作的负载均衡和简单的故障恢复。缺陷是**故障恢复无法自动化**、**写操作无法负载均衡**、存储能力受到**单机**的限制。
  
 3.  **<font color="#ab2e2e">哨兵：</font>** 在复制的基础上，哨兵实现了**自动化的故障恢复**。缺陷是 **写操作无法负载均衡**，存储能力受到**单机**的限制。
 
 4.   **<font color="#ab2e2e">集群：</font>** 通过集群实现了 **写操作负载均衡**，**自动化的故障恢复**， 存储能力**不受到单机限制** ， 是较为完善的高可用方案。


### 二、Redis Sentinel 深入探究

#### **2.1 Redis Sentinel的架构**

![image.png-97.1kB][1]
 

#### **2.2  Redis Sentinel的主要功能**

Sentinel 的主要功能包括 **主节点存活检测**、**主从运行情况检测**、**自动故障转移** 。
Redis的 **Sentinel最小配置** 是 **一主一从**。

Redis的Sentinel 可以执行以下**四个任务**：

 **<font color="#af684d">1.监控</font>**

Sentinel 会不断的 **检查** **主服务器** 和 **从服务器** 是否正常运行。
  

 **<font color="#af684d">2.通知</font>**
 
 当被监控的某个 **Redis服务器出现问题**，Sentinel 通过 **API 脚本** 向 **管理员** 或者其他的 **应用程序** 发送**通知**。
 
 
 **<font color="#af684d">3.自动故障转移</font>**
 
 当 主节点 不能正常工作时，Sentinel 会开始一次 **自动的** **故障转移**操作，它会将与 失效主节点 是 主从关系 的其中一个 **从节点** 升级为 **新的主节点**，并且将其他的 从节点 指向 新的主节点。
 
 
 **<font color="#af684d">4.配置提供者</font>**
 
 在 Redis Sentinel 模式下，**客户端应用** 在初始化时连接的是 Sentinel **节点集合**，从中 **获取** **主节点** 的信息。
 
#### **2.3  主观下线和客观下线**
 
 默认情况下，**每个Sentinel节点** 会以 **每秒一次** 的频率对 **Redis节点** 和 **其它的Sentinel**节点发送 **PING** 命令，并通过节点的 回复 来判断节点是否在线。
 
 **<font color="#af684d">1.主观下线</font>**
 
 主观下线 适用于所有 主节点 和 从节点。如果在 **down-after-milliseconds 毫秒内**，Sentinel **没有收到** 目标节点的 **有效回复**，则会判定 该节点 为 主观下线。
 
 
  **<font color="#af684d">2.客观下线</font>**
 
 客观下线 只适用于 **主节点**。如果 主节点 **出现故障**，Sentinel 节点会通过 **sentinel is-master-down-by-addr** 命令，向其它 Sentinel 节点 **询问** 对该节点的 **状态判断**。如果超过 **quorum** 个数的节点判定 主节点 不可达，则该 Sentinel 节点会判断 主节点 为 客观下线。


#### **2.4 Sentinel的通信命令**

Sentinel 节点连接一个 Redis 实例的时候，会创建 **cmd** 和 **pub/sub** 两个 连接。Sentinel 通过 **cmd** **连接**给 **Redis** 发送命令，通过 **pub/sub** **连接**到 Redis 实例上的其他 **Sentinel** 实例。

> **Sentinel** 与 **Redis** 主节点 和 从节点 交互的 **命令**，主要包括：


**<font color="#ab2e2e">ping：</font>**  Sentinel 向 Redis 节点发送 PING 命令，**检查** 节点的 **状态**

 **<font color="#ab2e2e">info：</font>**  Sentinel 向 Redis 节点发送 INFO 命令，获取它的 **从节点**信息
 
 **<font color="#ab2e2e">PUBLISH：</font>** Sentinel 向其监控的 Redis 节点 __sentinel__:hello 这个 channel 发布 **自己的信息** 及 **主节点** 相关的**配置**
 
 **<font color="#ab2e2e">SUBSCRIBE：</font>** Sentinel 通过 **订阅**  Redis 主节点 和 从节点 的 __sentinel__:hello 这个 channnel，获取正在 **监控相同服务** 的其他 **Sentinel节点**
 
 > **Sentinel** 与 **Sentinel** 交互的 **命令**，主要包括：
 
 **<font color="#ab2e2e">ping：</font>** Sentinel 向其他 Sentinel 节点发送 PING 命令，检查节点的状态
 
  **<font color="#ab2e2e">SENTINEL:is-master-down-by-addr：</font>** 和其他 Sentinel **协商** **主节点 的状态**，如果 主节点 处于 **SDOWN** 状态，则 **投票** 自动选出 **新的主节点**
 
 
#### **2.5 Sentinel的通信命令**

 每个 Sentinel 节点都需要 **定期执行** 以下任务：
 
> *  每个 Sentinel 以 **每秒钟** 一次的频率，向它所知的 **主服务器**、**从服务器** 以及其他 **Sentinel** 实例 发送一个 **PING** 命令。
 
 ![image.png-170.3kB][2]
 
 
 > * 如果一个 实例（instance）距离 最后一次 有效回复 PING 命令的 **时间** 超过 **down-after-milliseconds** 所指定的值，那么这个实例会被 Sentinel 标记为 **主观下线**。
 
![image.png-182.4kB][3]

> * 如果一个 主服务器 被标记为 **主观下线**，那么正在 监视 这个 主服务器 的所有 Sentinel 节点，要以 **每秒一次** 的频率确认 主服务器 的确进入了 主观下线 状态。

![image.png-190.9kB][4]

> *  如果一个 主服务器 被标记为 **主观下线**，并且有 **足够数量** 的 **Sentinel**（至少要达到 配置文件 指定的数量）在指定的 **时间范围** 内同意这一判断，那么这个 主服务器 被标记为 **客观下线**。

![image.png-173.2kB][5]


> * 在一般情况下， 每个 Sentinel 会以每 **10 秒**一次的频率，向它已知的所有 **主服务器** 和 **从服务器 发送 INFO 命令**。当一个 主服务器 被 Sentinel 标记为 **客观下线** 时，Sentinel 向 下线主服务器 的所有 从服务器 发送 INFO 命令的频率，会从 **10 秒一次**改为 **1秒一次**。

![image.png-168.2kB][6]


> * Sentinel 和其他 Sentinel **协商** 主节点 的状态，如果 **主节点** 处于 **SDOWN** 状态，则 **投票** 自动选出新的 **主节点**。将剩余的 从节点 指向 新的主节点 进行 数据复制。

![image.png-166.7kB][7]


> * 当没有足够数量的 Sentinel 同意 主服务器 下线时， 主服务器 的 主观下线状态 就会被移除。当 主服务器 重新向 Sentinel 的 PING 命令返回 有效回复 时，主服务器 的 **主观下线状态** 就会被 **移除**。

![image.png-75kB][8]



### 三、Redis Sentinel搭建

#### **3.1  Redis Sentinel的部署须知**

 1. 一个**稳健的** Redis Sentinel 集群，应该使用 至少 **三个 Sentinel 实例**，并且保证讲这些实例放到 **不同的机器** 上，甚至不同的 **物理区域**。
 
 2. Sentinel 无法保证 **强一致性。**
 
 3. 常见的 **客户端应用库** 都支持 Sentinel
 
 4. Sentinel 需要通过不断的 **测试** 和 **观察**，才能保证高可用。
 
 
 

#### **3.2  Redis Sentinel的节点规划** (部署在六个虚拟机中)
![image.png-16.6kB][9]



##### **3.2.1** 分别修改 **redis-server** 主从关系三份 **conf** 配置文件：

> 主节点

```java
daemonize yes
pidfile /var/run/redis_6379.pid
logfile /var/log/redis/redis_6379.log
port 6379
bind 0.0.0.0
timeout 300
databases 16
dbfilename dump.rdb
dir /usr/local/redis-5.0.7
masterauth redis123
requirepass redis123
```

> 从节点1 ; 从节点2

```java
daemonize yes
pidfile /var/run/redis_6379.pid
logfile /var/log/redis/redis_6379.log
port 6379
bind 0.0.0.0
timeout 300
databases 16
dbfilename dump.rdb
dir /usr/local/redis-5.0.7
masterauth redis123
requirepass redis123
replicaof 192.168.40.128 6379 
```

然后分别启动这三台redis-server，怎样开启已经怎样设置开启启动，请看我之前的文章介绍，这里不做详解。



##### **3.2.2** **Sentinel** 的 **配置** 以及设置 **开机启动**

 1. 复制sentinel.conf 到 /etc/redis 目录下 

  > cp /usr/local/redis-5.0.7/sentinel.conf /etc/redis/26379.conf
 
 

 2. 编辑26379.conf 文件
 
 ```java
 protected-mode no
 bind 0.0.0.0
 port 26379
 daemonize yes
 sentinel monitor mymaster 192.168.40.128 6379 2
 sentinel down-after-milliseconds mymaster 5000
 sentinel failover-timeout mymaster 180000
 sentinel parallel-syncs mymaster 1
 sentinel auth-pass mymaster redis123
 logfile /var/log/redis/sentinel-26379.log
 pidfile /var/run/redis-sentinel.pid
 ```
 
 3. 将redis的启动脚本复制一份放到/etc/init.d目录下
 
  > cp /usr/local/redis-5.0.7/utils/redis_init_script /etc/init.d/sentinel

 4. 编辑 sentinel 文件，把红框里面的内容改为sentinel相应的一些地址
 ![image.png-27.9kB][10]
 
 5. 增加sentinel的访问权限
   > chmod 777 /etc/init.d/sentinel

 6. 设置开机启动
 
   > chkconfig sentinel on
 
 7. sentinel 服务的开启和关闭
    > service sentinel start
    > service sentinel stop
 

 8. 开启 26379 端口，防止防火墙
 

##### **3.2.3** **Sentinel的启动** 验证

分别启动三台sentinel服务器

> 节点 192.168.40.131:26379

查看 /var/log/redis/sentinel-26379.log

![image.png-20.9kB][11]

从截图中可以看出当前的sentinel加入sentinel集群时，会有一个唯一的ID 去标识，并且展示了 监控的主节点 以及两个从节点的信息。

同理：启动另外两台上面的sentinel服务时，亦可看到同样的信息。

##### **3.2.4 Sentinel的配置刷新** 
> 节点 192.168.40.131:26379

打开 其中一台sentinel 服务的配置文件，也就是上文所说的26379.conf 文件，拉到文件最后面，可以看到自动追加了一些信息，包含了两个从节点的ip+port信息，以及其他的sentinel的信息
![image.png-16.6kB][12]

同理：我们在打开其他机器上的sentinel配置文件时，亦可以看到类似的信息。

##### **3.2.5 Sentinel的命令操作** 

> sentinel master (master_name)   //显示指定 主节点 的信息和状态。

![image.png-20.2kB][13]

>  sentinel slaves (master_name) //查看mymaster的从信息,这里可以看到两个从节点的信息

![image.png-9.5kB][14]
![image.png-7.1kB][15]


> sentinel sentinels (master_name)  //查看其它sentinel信息

![image.png-9.8kB][16]
![image.png-8.3kB][17]

> sentinel get-master-addr-by-name （master_name） //返回指定 主节点 的 IP 地址和 端口。

![image.png-3.2kB][18]



### 四、Redis 的故障切换与恢复

#### 4.1 我们先把 192.168.40.128  的redis服务停掉，也就是主节点服务先停掉。

![image.png-6.8kB][19]

#### 4.2这个时候我们再执行 sentinel get-master-addr-by-name 命令。从截图中可以看出 redis的主节点变成了192.168.40.129

![image.png-4.4kB][20]

#### 4.3 我们再执行 sentinel slaves 命令，就可以看到 从节点信息变成了 192.168.40.128 以及 192.168.40.130
     只不过 192.168.40.128 的 runid 为空，当前为磐机状态
   ![image.png-8.2kB][21]
   
#### 4.4 查看 redis.conf 的配置
    
    1.  192.168.40.129 的配置 replicaof 的信息已去除
    
    2.  192.168.40.130 的配置 replicaof 变成了 replicaof 192.168.40.129 6379
    
#### 4.5 重启 192.168.40.128 节点的redis服务， 执行 sentinel 命令 然后再查看配置信息，发现128 目前作为从节点并且zheng
   
    
 ![image.png-2.9kB][22]
    
![image.png-9.7kB][23]
    
 在128服务redis配置文件的最下面自动追加了replicaof 192.168.40.129 6379 信息
    ![image.png-3.3kB][24]
    
    
    
    
    
    


  [1]: http://static.zybuluo.com/qxjbeyond/3j0dyfo8spckout5ml1q42i5/image.png
  [2]: http://static.zybuluo.com/qxjbeyond/33ou2rryjj847gl3gfchkm73/image.png
  [3]: http://static.zybuluo.com/qxjbeyond/clr8q6rjoyk000bukf61vm2j/image.png
  [4]: http://static.zybuluo.com/qxjbeyond/ng3b1br2ttmggxcoli91xyh5/image.png
  [5]: http://static.zybuluo.com/qxjbeyond/vhb2ruc53xqrm1z25u0yahps/image.png
  [6]: http://static.zybuluo.com/qxjbeyond/qnbuze5tt7acgd8b3b9dkpjz/image.png
  [7]: http://static.zybuluo.com/qxjbeyond/xx12c9uq49gnkqyluybpualg/image.png
  [8]: http://static.zybuluo.com/qxjbeyond/h1cy1302m2rh7jlpn5ojhtfy/image.png
  [9]: http://static.zybuluo.com/qxjbeyond/39cjbyib8uuihwlbod3zkoiu/image.png
  [10]: http://static.zybuluo.com/qxjbeyond/16ct2ux2m3xseevw12rch4e2/image.png
  [11]: http://static.zybuluo.com/qxjbeyond/fsie3i2ala4f7h987hdvq7qn/image.png
  [12]: http://static.zybuluo.com/qxjbeyond/jvv9uey7yn0z2sg975f9koub/image.png
  [13]: http://static.zybuluo.com/qxjbeyond/vi4hk380rmcdx8ntzk27bdos/image.png
  [14]: http://static.zybuluo.com/qxjbeyond/85y1ml9f93b6ihppu9ft1ybj/image.png
  [15]: http://static.zybuluo.com/qxjbeyond/1gpe3ns37bwrqrrgh932l9pd/image.png
  [16]: http://static.zybuluo.com/qxjbeyond/f8hj9reu0echz94mipvxlr9l/image.png
  [17]: http://static.zybuluo.com/qxjbeyond/sftphji3djf2ae910rlp2on7/image.png
  [18]: http://static.zybuluo.com/qxjbeyond/6ozwt9jtbc41ee5zi3bs5rq3/image.png
  [19]: http://static.zybuluo.com/qxjbeyond/caokk1l08kwr37t3tjzxzxuc/image.png
  [20]: http://static.zybuluo.com/qxjbeyond/34pr0xovjlo464nfj2ho5l9e/image.png
  [21]: http://static.zybuluo.com/qxjbeyond/efkq8q38jmv7o88gqfeyc3cy/image.png
  [22]: http://static.zybuluo.com/qxjbeyond/7dt5dag4anoo1223kxarpbt1/image.png
  [23]: http://static.zybuluo.com/qxjbeyond/krfpiq23ikhhmz5dugjs5d6b/image.png
  [24]: http://static.zybuluo.com/qxjbeyond/756cgx4qc4kqxooi6qzeywnl/image.png
