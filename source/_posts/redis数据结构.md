---
title: redis数据结构
date: 2023-10-11 20:55:55
tags: redis
---

# 全局哈希表
一维数组 + 二维链表
hash(key) % 数组长度
时间复杂度O(1)
 {% asset_img image-1.png %}

## 1.string类型
应用：计数器/分布式锁/对象缓存
#### 1.int 存整形
#### 2.SDS： 字符串是一个字符串值并且长度大于44个字节 简单动态字符串，encoding设置为raw
```
/*  
 * 保存字符串对象的结构  
 */  
struct sdshdr {  
    int len;  // buf 中已占用空间的长度  
    int free;  // buf 中剩余可用空间的长度
    char buf[];  // 数据空间  
};

```
SDS 相比C 字符串的优势：

SDS保存了字符串的长度，而C字符串不保存长度，需要遍历整个数组（找到’\0’为止）才能取到字符串长度。
修改SDS时，检查给定SDS空间是否足够，如果不够会先拓展SDS 的空间，防止缓冲区溢出。C字符串不会检查字符串空间是否足够，调用一些函数时很容易造成缓冲区溢出（比如strcat字符串连接函数）。
SDS预分配空间的机制，可以减少为字符串重新分配空间的次数。空间预分配用于优化 SDS 的字符串增长操作
惰性空间释放用于优化 SDS 的字符串缩短操作，程序不会立即使用内存重分配来回收缩短后多出来的字节，而是使用 free 属性将这个字节的数量记录起来，并等待将来使用

#### 3.embstr：字符串长度小于等于44个字节，encoding设置为embstr

## 2.Hash类型
应用：对象信息（电商购物车）(对象各属性存到value又是key-value的形式)/高并发场景下使用Redis生成唯一的id
#### 1.字典,hashtable
```
typedef struct dictht {
   dictEntry **table;//哈希表数组   
   unsigned long size;//哈希表大小  
   unsigned long sizemask;//哈希表大小掩码，用于计算索引值
   unsigned long used;//该哈希表已有节点的数量
}
```
hashtable中计算出hash值后，还要通过哈希表大小掩码sizemask 属性和哈希值再次得到数组下标。
解决哈希冲突：假如hashtable中不同的key通过计算得到同一个index，就会形成单向链表（链地址法又叫拉链法）
渐进式rehash
#### 2.ziplist 压缩列表 （当哈希类型元素个数小于hash-max-ziplist-entries配置，默认512个 ，所有值都小于hash-max-ziplist-value配置，默认64字节）
压缩列表（ziplist）是一组连续内存块组成的顺序的数据结构，压缩列表能够节省空间，压缩列表中使用多个节点来存储数据。
 {% asset_img image-2.png %}

- zlbytes：4个字节的大小，记录压缩列表占用内存的字节数。
- zltail：4个字节大小，记录表尾节点距离起始地址的偏移量，用于快速定位到尾节点的地址。
- zllen：2个字节的大小，记录压缩列表中的节点数。
- entry：表示列表中的每一个节点。
    每一个entry节点又有三部分组成，包括previous_entry_ength、encoding、content。
    - previous_entry_ength表示前一个节点entry的长度，可用于计算前一个节点的真实地址，因为他们的地址是连续的。
    - encoding：这里保存的是content的内容类型和长度。
    - content：content保存的是每一个节点的内容。
- zlend：表示压缩列表的特殊结束符号'0xFF'。

## 3.List类型
应用：阻塞队列/栈
#### 1.zipList （元素个数小于zset-max-ziplist-entries配置，默认128，所有值都小于zset-max-ziplist-value配置，默认64）
同上
#### 2.双向链表
 {% asset_img image-3.png %}
Redis中链表的特性：
每一个节点都有指向前一个节点和后一个节点的指针。
头节点和尾节点的prev和next指针指向为null，所以链表是无环的。
链表有自己长度的信息，获取长度的时间复杂度为O(1)。

## 4.Set集合
应用：去重、抽奖、共同好友、二度好友
#### 1.hashtable
同上
#### 2.intset （Set集合中必须是64位有符号的十进制整型且元素个数不超过set-max-intset-entries配置，默认512）
inset也叫做整数集合，用于保存整数值的数据结构类型，它可以保存int16_t、int32_t 或者int64_t 的整数值
有三个属性值encoding、length、contents[]，分别表示编码方式、整数集合的长度、以及元素内容，length就是记录contents里面的大小。
在整数集合新增元素的时候，若是超出了原集合的长度大小，就会对集合进行升级，具体的升级过程如下：

首先扩展底层数组的大小，并且数组的类型为新元素的类型。
然后将原来的数组中的元素转为新元素的类型，并放到扩展后数组对应的位置。
整数集合升级后就不会再降级，编码会一直保持升级后的状态。

## 5.zset集合
应用：排行榜
#### 1.ziplist（数据少时用）
ziplist为了节省内存，每个元素占用的空间可以不同，对于大的数据（long long），就多用一些字节来存储，而对于小的数据（short），就少用一些字节来存储。因此查找的时候需要按顺序遍历。ziplist省内存但是查找效率低。
#### 2.字典+skiplist （数据多时用）
 {% asset_img image-3.png %}
有序单链表构造的，通过构建索引提高查找效率，空间换时间，查找方式是从最上面的链表层层往下查找，最后在最底层的链表找到对应的节点
除了最底层的一层保存的是原始链表的完整数据，上层的节点数会越来越少，并且跨度会越来越大

## 6.3种特殊数据结构 ：HyperLogLogs（基数统计）、Bitmap （位存储）、Geospatial (地理位置)
