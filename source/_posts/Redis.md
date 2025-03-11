---
title: redis
date: 2022-02-17 21:09:31
tags: redis
categories: 编程
---
# Redis

## 描述

REmote DIctionary Server(Redis) 是 key-value 存储系统，是跨平台的非关系型数据库。

redis基于内存操作，cpu不是redis性能瓶颈，瓶颈是根据机器的内存和网络带宽，单线程。多线程会产生cpu上下文的切换，耗时

 {% asset_img image-1.png %}



## 优势

**高性能**

假如用户第一次访问数据库中的某些数据的话，这个过程是比较慢，毕竟是从硬盘中读取的。但是，如果说，用户访问的数据属于高频数据并且不会经常改变的话，用户下一次再访问这些数据的时候就可以直接从缓存中获取了。操作缓存就是直接操作内存，所以速度相当快。

**高并发**

一般像 MySQL 这类的数据库的 QPS 大概都在 1w 左右（4 核 8g） ，但是使用 Redis 缓存之后很容易达到 10w+，甚至最高能达到 30w+（就单机 Redis 的情况，Redis 集群的话会更高）。

> QPS（Query Per Second）：服务器每秒可以执行的查询次数



## Redis 数据结构

- **5 种基础数据结构** ：String（字符串）、List（列表）、Set（集合）、Hash（散列）、Zset（有序集合）。

  - String

  {% asset_img image-2.png %}

  {% asset_img image-3.png %}

  {% asset_img image-4.png %}

  {% asset_img image-5.png %}

  {% asset_img image-6.png %}

  {% asset_img image-7.png %}

  {% asset_img image-8.png %}

  {% asset_img image-9.png %}

  - List

   {% asset_img image-10.png %}

   {% asset_img image-11.png %}

   {% asset_img image-12.png %}

   {% asset_img image-13.png %}

   {% asset_img image-14.png %}

   {% asset_img image-15.png %}

   {% asset_img image-16.png %}

   {% asset_img image-17.png %}

   {% asset_img image-18.png %}

  - Set

   {% asset_img image-19.png %}
   {% asset_img image-20.png %}
   {% asset_img image-21.png %}
   {% asset_img image-22.png %}
   {% asset_img image-23.png %}
   {% asset_img image-24.png %}

  - Hash

   {% asset_img image-25.png %}
   {% asset_img image-26.png %}
    ![image-20220923193049607](C:\Users\yangz\AppData\Roaming\Typora\typora-user-images\image-20220923193049607.png)

{% asset_img image-27.png %}
{% asset_img image-28.png %}
  - Zset（有序集合）

{% asset_img image-29.png %}
{% asset_img image-30.png %}
{% asset_img image-31.png %}
{% asset_img image-32.png %}

- **3 种特殊数据结构** ：HyperLogLogs（基数统计）、Bitmap （位存储）、Geospatial (地理位置)。

  - Geospatial 

  {% asset_img image-33.png %}
  {% asset_img image-34.png %}
  {% asset_img image-35.png %}
  {% asset_img image-36.png %}
  {% asset_img image-37.png %}
  {% asset_img image-38.png %}

  - Hyperloglog基数

    优点：查找不重复的元素，占内存小，12k内存，可以用于统计网页uv，比传统set方式节省空间

      {% asset_img image-39.png %}
        {% asset_img image-40.png %}

  - Bitmaps位图

  {% asset_img image-41.png %}
  {% asset_img image-42.png %}
  {% asset_img image-43.png %}
    

## 性能测试

1. redis-benchmark -h localhost -p 6379 -c 100 -n 100000

​		-c 100 100个并发连接

​		-n 100000 每个连接100000个请求

​	![image-20220921230507325](C:\Users\yangz\AppData\Roaming\Typora\typora-user-images\image-20220921230507325.png)

2. 分析

![image-20220921230332185](C:\Users\yangz\AppData\Roaming\Typora\typora-user-images\image-20220921230332185.png)



## 事务

- 一组命令的集合。一个事务中的所有命令会被序列化。事务执行时按顺序执行。redis单条命令是保证原子性的，但是事务不保证原子性，没有隔离级别的概念。

- 三个阶段

  - 开启事务
  - 命令入队
  - 执行事务

  ```bash
  multi
  set k1 v1
  set k2 v2
  get k1
  exec
  ```

  - 取消事务 `dicard`

- 异常

  - 编译型异常

      {% asset_img image-44.png %}

  - 运行时异常

      {% asset_img image-45.png %}
- 乐观锁，监视测试

  - 获取version
  - 更新的时候比较version
  - 事务执行失败，解锁unwatch

      {% asset_img image-46.png %}
      {% asset_img image-47.png %}


## Jedis

1. 依赖

   ```java
       <dependencies>
           <!--导入Jedis的包-->
           <dependency>
               <groupId>redis.clients</groupId>
               <artifactId>jedis</artifactId>
               <version>4.2.3</version>
           </dependency>
           <!--导入fastjson的包-->
           <dependency>
               <groupId>com.alibaba</groupId>
               <artifactId>fastjson</artifactId>
               <version>2.0.13</version>
           </dependency>
       </dependencies>
   ```

2. 测试ping

   ```java
   public class TestPingRedis {
       public static void main(String[] args) {
           Jedis jedis = new Jedis("127.0.0.1",6379);
           System.out.println(jedis.ping());
       }
   }
   ```

3. 测试事务

   ```java
       public static void main(String[] args) {
           Jedis jedis = new Jedis("127.0.0.1",6379);
           jedis.flushDB();
           JSONObject jsonObject = new JSONObject();
           jsonObject.put("a","one");
           jsonObject.put("b","two");
           jsonObject.put("c","three");
           // 开启事务
           Transaction transaction = jedis.multi();
           try{
               transaction.set("json1",jsonObject.toString());
               transaction.set("json2",jsonObject.toString());
               transaction.exec();
           }
           catch(Exception e){
               transaction.discard();
               e.printStackTrace();
           }finally {
               System.out.println(jedis.get("json1"));
               System.out.println(jedis.get("json2"));
               jedis.close();
           }
       }
   ```

   

## 整合SpringBoot

springboot 2.x后，jedis被替换为了lettuce。jedis多个线程操作不安全，jedis pool（BIO）。lettuce采用netty，可以在多个线程中共享（NIO）

      {% asset_img image-48.png %}


Redis配置文件

- network

  ```bash
  bind 127.0.0.1 #绑定ip
  protected-mode yes #保护模式
  port 6379 #端口
  ```

- general

  ```bash
  daemonize yes #守护进程的方式运行，默认是no
  pidfile /var/run/redis_6379.pid #指定后台运行的pid文件
  loglevel notice # 日志等级
  logfile "" # 日志文件位置
  databases 16 # 数据库数量，默认16个
  ```

- 快照

  ```bash
  save 3600 100 # 规定时间内，做了多少次操作会持久化到文件.rdb .aof
  stop-writes-on-bgsave-error yes # 持久化出错是否继续工作
  rdbcompression yes # 是否压缩rdb文件
  rdbchecksum yes # 保存rdb文件的时候错误校验
  dir ./ # rdb文件的位置
  ```

- 安全

  ```
  requirepass password # 设置密码，命令config set requirepass "password"
  ```

- 限制

  ```bash
  maxclients 1000 # 设置连接redis客户端最大数量
  maxmemory <bytes> # redis配置最大的内存容量
  maxmemory-policy noeviction # 内存到达上限的处理策略
  ```

  - maxmemory-policy

    1. volatile-lru：只对设置了过期时间的key进行LRU（默认值） 

    2. allkeys-lru ： 删除lru算法的key  

    3. volatile-random：随机删除即将过期key  

    4. allkeys-random：随机删除  

    5. volatile-ttl ： 删除即将过期的  

    6. noeviction ： 永不过期，返回错误

- Append Only模式 aof配置

  ```bash
  appendonly no # 默认不开启aof，默认使用rdb方式持久化
  appendfsync everysec # 每秒执行一次sync，可能丢失1s的数据
  appendfsync always # 每次修改执行sync，消耗性能
  appendfsync no # 不执行sync，操作系统自己同步
  ```



## 持久化RDB（redis database）

      {% asset_img image-49.png %}
- 描述：

  ​		Redis会单独创建（fork）一个子进程来进行持久化，会先将数据写入到一个临时文件中，待持久化过程都结束了。再用这个临时文件替换上次持久化好的文件。整个过程中，主进程是不进行任何IO操作的，这就确保了极高的性能。如果需要大规模数据的恢复，且对于数据恢复的完整性不是非常敏感，那RDB方式要比AOF更加高效，RDB的缺点是最后一次持久化后的数据可能丢失。

- 触发机制

  - save规则满足，自动触发
  - 执行flushall命令
  - 退出redis

- 恢复rdb文件

  1. 将rdb文件放在redis启动目录，redis启动自动检查
  2. config get dir 得到启动目录

- 优点

  - 适合大规模的数据恢复

  - 对数据的完整性要求不高

- 缺点
  - 需要一定的时间间隔进行操作，如果redis意外宕机，最后一次修改的数据会丢失
  - fork进程占用一定的内存空间



## 持久化AOF（Append Only File）

- 描述

  将所有的命令记录下来放入history文件，恢复的时候执行此文件。以日志的形式来记录每个写的操作，将Redis执行过的所有指令记录下来（读操作不记录），只许追加文件，但是不可以改写文件，redis启动之初会读取该文件重新构建数据，换言之，redis重启的话就根据日志文件的内容将写指令从前到后执行一次以完成数据的恢复工作

{% asset_img image-50.png %}
- 优点

  - 每次修改都同步，文件的完整性好
  - 每秒同步，可能会丢失一秒的数据
  - 从不同步效率最高

- 缺点

  - aof文件数据较大，修复速度慢
  - aof运行效率慢



## Redis发布订阅

原理;

通过SUBSCRIBE命令订阅某频道后，redis-server里维护了一个字典，字典的键就是一个个频道!，而字典的值则是一个链表，链表中保存了所有订阅这个channel的客户端。SUBSCRIBE命令的关键，就是将客户端添加到给定channel的订阅链表中。

通过PUBLSH命令向订阅者发送消息，redis-server会使用给定的频道作为键，在它所维护的channel字典中查找记录了订阅这个频道的所有客户端的链表，遍历这个链表，将消息发布给所有订阅者。



## Redis主从复制

- 描述：

  主从复制，是指将一台Redis服务器的数据，复制到其他的Redis服务器。前者称为主节点(masterleader)，后者称为从节点(slave/follower);数据的复制是单向的，只能由主节点到从节点。Master以写为主，Slave以读为主。

- 作用：

  - 数据冗余：实现了数据的热备份
  - 故障恢复：主节点出现故障，从节点提供服务
  - 负载均衡：在主从复制的基础上，配合读写分离，分担服务器负载，提高服务器的并发量
  - 高可用集群的基础

- 查看当前库的信息：info-replication

{% asset_img image-51.png %}
- 操作步骤

  1. 复制配置文件
  2. 修改端口、pid、log文件名字、dump.rdb名字

{% asset_img image-52.png %}
- 配置从机

  ```bash
  slaveof 127.0.0.1 6379 # 从属于主机的地址
  ```

{% asset_img image-53.png %}
- 复制原理

  Slave启动成功连接到master后会发送一个sync命令
  Master接到命令，启动后台的存盘进程，同时收集所有接收到的用于修改数据集命令，在后台进程执行完毕之后，master将传送整个数据文件到slave，并完成一次完全同步。

  - 全量复制：slave服务在接收到数据库文件数据后，将其存盘并加载到内存中。
  - 增量复制：Master继续将新的所有收集到的修改命令依次传给slave，完成同步。

{% asset_img image-54.png %}
- salveof no one

  如果主机断开了连接，可以使用这个命令来让自己成为主节点，其他的结点就可以手动连接到最新的这个主节点。如果这个时候主机修复了，那就只能重新连接



## 哨兵模式（Sentinel）

- 原理

  Redis提供了哨兵的命令，哨兵是一个独立的进程，作为进程，它会独立运行。其原理是**哨兵通过发送命令，等待Redis服务器响应，从而监控运行的多个Redis实例**

{% asset_img image-55.png %}
- 作用

  1. 通过发送命令，让Redis服务器返回监控其运行状态，包括主服务器和从服务器
  2. 当哨兵监测到master宕机，会自动将slave切换成master，然后通过发布订阅模式通知其他的从服务器，修改配置文件，让他们切换主机

- 多哨兵模式

  一个哨兵进程对Redis服务器进行监控，可能会出现问题，为此，我们可以使用多个哨兵进行监控。各个哨兵之间还会进行监控，这样就形成了多哨兵模式。

  哨兵1先检测到主机宕机，系统并不会马上进行failover过程，仅仅是哨兵1主观的认为主服务器不可用，这个现象成为主观下线。当后面的哨兵也检测到主服务器不可用，并且数量达到一定值时，那么哨兵之间就会进行一次投票，投票的结果由一个哨兵发起，进行failover[故障转移]操作。切换成功后，就会通过发布订阅模式，让各个哨兵把自己监控的从服务器实现切换主机，这个过程称为客观下线。如果主机此时回来了，只能归并到新的主机下，当做从机。

{% asset_img image-56.png %}
- 配置sentinel.conf

{% asset_img image-57.png %}
- 启动哨兵

  ```bash
  redis-sentinel ../etc/sentinal.conf
  ```

- 优点

  1. 哨兵模式，基于主从复制模式，所有主从配置的优点它都有
  2. 主从可以切换，故障可以转移，系统的可用性就会更好
  3. 哨兵模式就是主从模式的升级，手动到自动，更加健壮

- 缺点

  1. Redis不好在线扩容，集群容量一旦达到上限，在线扩容就十分麻烦
  2. 实现哨兵模式的配置是比较复杂

- 哨兵配置

  ```bash
  # Example sentinel.conf
   
  # 哨兵sentinel实例运行的端口 默认26379
  port 26379
   
  # 哨兵sentinel的工作目录
  dir /tmp
   
  # 哨兵sentinel监控的redis主节点的 ip port 
  # master-name  可以自己命名的主节点名字 只能由字母A-z、数字0-9 、这三个字符".-_"组成。
  # quorum 当这些quorum个数sentinel哨兵认为master主节点失联 那么这时 客观上认为主节点失联了
  # sentinel monitor <master-name> <ip> <redis-port> <quorum>
  sentinel monitor mymaster 127.0.0.1 6379 1
   
  # 当在Redis实例中开启了requirepass foobared 授权密码 这样所有连接Redis实例的客户端都要提供密码
  # 设置哨兵sentinel 连接主从的密码 注意必须为主从设置一样的验证密码
  # sentinel auth-pass <master-name> <password>
  sentinel auth-pass mymaster MySUPER--secret-0123passw0rd
   
   
  # 指定多少毫秒之后 主节点没有应答哨兵sentinel 此时 哨兵主观上认为主节点下线 默认30秒
  # sentinel down-after-milliseconds <master-name> <milliseconds>
  sentinel down-after-milliseconds mymaster 30000
   
  # 这个配置项指定了在发生failover主备切换时最多可以有多少个slave同时对新的master进行 同步，
  这个数字越小，完成failover所需的时间就越长，
  但是如果这个数字越大，就意味着越 多的slave因为replication而不可用。
  可以通过将这个值设为 1 来保证每次只有一个slave 处于不能处理命令请求的状态。
  # sentinel parallel-syncs <master-name> <numslaves>
  sentinel parallel-syncs mymaster 1
   
   
   
  # 故障转移的超时时间 failover-timeout 可以用在以下这些方面： 
  #1. 同一个sentinel对同一个master两次failover之间的间隔时间。
  #2. 当一个slave从一个错误的master那里同步数据开始计算时间。直到slave被纠正为向正确的master那里同步数据时。
  #3.当想要取消一个正在进行的failover所需要的时间。  
  #4.当进行failover时，配置所有slaves指向新的master所需的最大时间。不过，即使过了这个超时，slaves依然会被正确配置为指向master，但是就不按parallel-syncs所配置的规则来了
  # 默认三分钟
  # sentinel failover-timeout <master-name> <milliseconds>
  sentinel failover-timeout mymaster 180000
   
  # SCRIPTS EXECUTION
   
  #配置当某一事件发生时所需要执行的脚本，可以通过脚本来通知管理员，例如当系统运行不正常时发邮件通知相关人员。
  #对于脚本的运行结果有以下规则：
  #若脚本执行后返回1，那么该脚本稍后将会被再次执行，重复次数目前默认为10
  #若脚本执行后返回2，或者比2更高的一个返回值，脚本将不会重复执行。
  #如果脚本在执行过程中由于收到系统中断信号被终止了，则同返回值为1时的行为相同。
  #一个脚本的最大执行时间为60s，如果超过这个时间，脚本将会被一个SIGKILL信号终止，之后重新执行。
   
  #通知型脚本:当sentinel有任何警告级别的事件发生时（比如说redis实例的主观失效和客观失效等等），将会去调用这个脚本，
  #这时这个脚本应该通过邮件，SMS等方式去通知系统管理员关于系统不正常运行的信息。调用该脚本时，将传给脚本两个参数，
  #一个是事件的类型，
  #一个是事件的描述。
  #如果sentinel.conf配置文件中配置了这个脚本路径，那么必须保证这个脚本存在于这个路径，并且是可执行的，否则sentinel无法正常启动成功。
  #通知脚本
  # sentinel notification-script <master-name> <script-path>
    sentinel notification-script mymaster /var/redis/notify.sh
   
  # 客户端重新配置主节点参数脚本
  # 当一个master由于failover而发生改变时，这个脚本将会被调用，通知相关的客户端关于master地址已经发生改变的信息。
  # 以下参数将会在调用脚本时传给脚本:
  # <master-name> <role> <state> <from-ip> <from-port> <to-ip> <to-port>
  # 目前<state>总是“failover”,
  # <role>是“leader”或者“observer”中的一个。 
  # 参数 from-ip, from-port, to-ip, to-port是用来和旧的master和新的master(即旧的slave)通信的
  # 这个脚本应该是通用的，能被多次调用，不是针对性的。
  # sentinel client-reconfig-script <master-name> <script-path>
  sentinel client-reconfig-script mymaster /var/redis/reconfig.sh
  ```

  

## Redis缓存穿透

- 描述：

  用户想要查询一个数据，发现redis内存数据库没有，也就是缓存没有命中，于是向持久层数据库查询。发现也没有，于是本次查询失败。当用户很多的时候，缓存都没有命中，于是都去请求了持久层数据库。这会给持久层数据库造成很大的压力，这时候就相当于出现了缓存穿透。

- 方案

  - 布隆过滤器

    布隆过滤器是一种数据结构，对所有可能查询的参数以hash形式存储，在控制层先进行校验，不符合则丢弃，从而避免了对底层存储系统的查询压力

{% asset_img image-58.png %}
  - 缓存空对象

    缓存中存储较多无意义的空对象

    缓存层和存储层的数据可能会出现数据不一致的情况



## 缓存击穿

- 描述：缓存击穿，是指一个key非常热点，在不停的扛着大并发，大并发集中对这一个点进行访问，当这个key在失效的瞬间，持续的大并发就穿破缓存，直接请求数据库，并且回写缓存，会导使数据库瞬间压力过大。

- 解决方案

  - 设置热点数据永不过期

    从缓存层面来看，没有设置过期时间，所以不会出现热点key过期后产生的问题、

  - 加互斥锁

    分布式锁：使用分布式锁，保证对于每个key同时只有一个线程去查询后端服务，其他线程没有获得分布式锁的权限，因此只需要等待即可。这种方式将高并发的压力转移到了分布式锁，因此对分布式锁考验很大



## 缓存雪崩

- 描述

  缓存雪崩，是指在某一个时间段，缓存集中过期失效，产生周期性的压力波峰。于是所有的请求都会达到存储层，存储层的调用量会暴增。Redis宕机。

- 解决方案

  - Redis高可用：多增设几台redis，这样一台挂掉之后其他的还可以继续工作，其实就是搭建的集群
  - 限流降级：在缓存失效后，通过加锁或者队列来控制读数据库写缓存的线程数量。比如对某个key只允许一个线程查询数据和写缓存，其他线程等待。
  - 数据预热：在正式部署之前，我先把可能的数据先预先访问一遍，这样部分可能大量访问的数据就会加载到缓存中。在即将发生大并发访问前手动触发加载缓存不同的key，设置不同的过期时间，让缓存失效的时间点尽量均匀。