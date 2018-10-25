---
title: redis源码阅读(零) 概述
date: 2018-08-01 12:23:19
tags:
- redis
categories:
- 源码阅读
---

> 历时3个月，粗浅的过了一遍源码，因为没有使用过集群，对于这部分看的比较随意，多是参考资料进行学习
>
> 对于其他一些功能源码并未阅读，等以后有兴趣了再看看

源码版本 redis-4.0.10

参考资料：

- https://github.com/huangz1990/redis-3.0-annotated
- https://github.com/menwengit/redis_source_annotation
- 《Redis设计与实现》第二版 —— 黄健宏



|          文件名称          |      作用      |
| :------------------------: | :------------: |
|       sds.c 和 sds.h       | 简单动态字符串 |
|    adlist.c 和 adlist.h    |      链表      |
|      dict.c 和 dict.h      |      字典      |
|    t_zset.c和 server.h     |     跳跃表     |
|    intset.c 和 intset.h    |    整数集合    |
|   ziplist.c 和 ziplist.h   |    压缩列表    |
|    zipmap.c 和 zipmap.h    |    压缩字典    |
| quicklist.c 和 quicklist.h |    快速列表    |
|    object.c 和 server.h    |    对象系统    |
|         t_string.c         |     字符键     |
|    t_list.c 和 server.h    |     列表键     |
|    t_hash.c 和 server.h    |     哈希键     |
|    t_set.c 和 server.h     |     集合键     |
|    t_zset.c 和 server.h    |   有序集合键   |
|      db.c 和 server.h      |  Redis数据库   |
|          notify.c          | 数据库通知功能 |
|       rdb.c 和 rdb.h       |   RDB持久化    |
|           aof.c            |   AOF持久化    |
|       rio.c 和 rio.h       |  抽象输入输出  |
|       replication.c        |  集群复制功能  |
|         sentinel.c         |  集群哨兵功能  |
|         cluster.c          |    集群功能    |

