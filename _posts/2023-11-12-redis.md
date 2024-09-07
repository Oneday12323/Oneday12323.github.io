# Remote dictionary server

> 开源基于内存的key-value数据存储系统，支持多种数据结构，支持持久化

用于缓存DB，消息队列Cache，分布式锁、分布式队列、主从复制

NOSql数据库，非关系型数据库
## 1. Redis 简介
### 1.1 Redis 是什么？
Redis 是完全开源免费的，遵守BSD协议，是一个高性能的key-value数据库。
Redis 与其他 key - value 缓存产品有以下三个特点：
- Redis支持数据的持久化，可以将内存中的数据保存在磁盘中，重启的时候可以再次加载进行使用。
- Redis不仅仅支持简单的key-value类型的数据，同时还提供list，set，zset，hash等数据结构的存储。
- Redis支持数据的备份，即master-slave模式的数据备份。
Redis 优势：
- 性能极高，读的速度是110000次/s,写的速度是81000次/s。
- 丰富的数据类型，支持string，list，set，zset，hash。
- 支持数据持久化，支持AOF和RDB两种持久化方式。
- 原子性，所有操作都是原子性的。
- 支持master-slave模式。
Redis 适用场景：
- 缓存，减轻数据库压力。
- 消息队列，比如任务队列。
- 分布式锁。
- 计数器，比如浏览量。
- 限速，比如限制每分钟请求数。
- 社交网络。
- 实时系统，比如活动排行榜。
- 实时分析，比如用户行为分析。
- 实时应用，比如即时聊天。
### 1.2 Redis 与 Memcached 的区别
- Redis支持数据持久化，而Memcached不支持。
- Redis支持数据结构，而Memcached不支持。
- Redis支持数据类型，而Memcached不支持。
- Redis支持数据复制，而Memcached不支持。
- Redis支持分布式，而Memcached不支持。
- Redis支持高可用，而Memcached不支持。
- Redis支持事务，而Memcached不支持。
- Redis支持管道，而Memcached不支持。
- Redis支持Lua脚本，而Memcached不支持。
- Redis支持多种编程语言，而Memcached只支持C语言。
### 1.3 Redis 数据类型
Redis支持五种数据类型：string（字符串），hash（哈希），list（列表），set（集合）及zset(sorted set
有序集合)。
### 1.4 Redis 优势
- 性能极高，读的速度是110000次/s,写的速度是81000次/s。和其他数据库对比，Redis有非常高的读写性能。
- 丰富的数据类型，支持string，list，set，zset，hash。单键值对最大支持512MB。
- 支持数据持久化，支持AOF和RDB两种持久化方式。
- 原子性，所有操作都是原子性的。
- 支持master-slave模式。
- 支持数据备份，即master-slave模式的数据备份。
- 丰富的特性：发布/订阅，key过期，持久化，事务，流水线，Lua脚本等等。
- 主从复制、哨兵模式高可用
### 1.5 Redis 应用场景
- 缓存，减轻数据库压力。
- 消息队列，比如任务队列。
- 分布式锁。
- 计数器，比如浏览量。
- 限速，比如限制每分钟请求数。
- 社交网络。
- 实时系统，比如活动排行榜。
- 实时分析，比如用户行为分析。
- 实时应用，比如即时聊天。
### 1.6 Redis 数据结构
Redis支持五种基本数据类型：string（字符串），hash（哈希），list（列表），set（集合）及zset(sorted set
有序集合)。
redis支持五种高级数据类型：位图bitmap，hyperloglog，地理空间geospatial，消息队列streams，位域Bitfield。
#### 1.6.1 String（字符串）
String是Redis最基本的类型，一个key对应一个value。
String类型是二进制安全的。意思是Redis的string可以包含任何数据。比如jpg图片或者序列化的对象。
String类型是Redis最基本的数据类型，一个key对应一个value。
#### 1.6.2 Hash（哈希）
Redis hash是一个键值对集合。
Redis hash是一个string类型的field和value的映射表，hash特别适合用于存储对象。
Redis 中每个hash可以存储40多亿个键值对。
#### 1.6.3 List（列表）
Redis列表是简单的字符串列表，按照插入顺序排序。你可以添加一个元素到列表的头部（左边）或者尾部（右边）。
Redis列表的实现为链表，这意味着list的插入和删除操作非常快，时间复杂度为O(1)。


早期互联网公司的应用系统，大多是通过MySQL这种传统数据库来对外提供服务的。互联网快速发展，应用系统的访问量逐渐变大，数据库性能瓶颈明显-因为磁盘IO导致的
因为磁盘IO的读写速度与内存相比是非常慢的，所以为了提高数据库性能，通常会使用缓存来缓解数据库压力。