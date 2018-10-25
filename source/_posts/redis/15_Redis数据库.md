---
title: Redis源码阅读(十五) 数据库对象
date: 2018-08-29 12:42:31
tags:
- redis
categories:
- 源码阅读
---

### 1. 定义

- Redis数据库结构

```c
typedef struct redisDb {
    // 键值对字典，保存数据库对象中的所有键值对
    dict *dict;                 /* The keyspace for this DB */

    // 过期字典, 保存设置了过期时间的键和过期时间
    dict *expires;              /* Timeout of keys with a timeout set */

    // 阻塞键字典，保存着造成客户端组队的键和被阻塞的客户端
    // key为阻塞的键，value为该阻塞键造成阻塞的客户端双向链表
    dict *blocking_keys;        /* Keys with clients waiting for data (BLPOP)*/

    // 阻塞键字典，保存着收到PUSH命令的键，避免重复操作
    dict *ready_keys;           /* Blocked keys that received a PUSH */

    // 监控键字典，保存着被WATCH命令监控的键
    dict *watched_keys;         /* WATCHED keys for MULTI/EXEC CAS */

    // 数据库ID
    int id;                     /* Database ID */

    // 平均生成周期，用于统计
    long long avg_ttl;          /* Average TTL, just for stats */
} redisDb;
```

- Redis服务器

  Redis服务器将所有数据库都保存在db数组中，通过dbnum来决定Redis数据库实例的个数。

```c
struct redisServer {
    // RedisDB对象数组，长度默认16，每个都存储了Redis数据库对象
    redisDb *db;
    
    // ...
   
    // 服务器db对象的数量，服务器初始化时根据dbnum确定
    int dbnum;
    
    // ...
}
```

- Redis客户端

```c
typedef struct client {
    // 当前客户端指向的数据库对象
    redisDb *db;
}
```

### 2. Redis数据库

#### 2.1 dict 键空间

redisDb结构中dict属性保存了该数据库所有的键值对，被称之为键空间。

- 每个键都是一个字符串，客户端所有设置key都存储在该字典中
- 每个值可以是字符串对象、列表对象、哈希对象、集合对象和有序集合对象中任意一种。

#### 2.2 expires 过期字典 

expires字典保存着当前数据库对象中键的过期时间

- 每个键都是一个指针，指向dict中过期的键
- 每个值都是一个long long类型的整数，保存了dict中key的过期时间(毫秒精度的时间戳)

#### 2.3 blocking_keys 阻塞键字典 

blocking_keys字典保存着造成客户端阻塞的键

- 每个键都是一个指针，指向造成客户端阻塞的dict中的键
- 每个值都是一个双端链表，存储着被这个键阻塞的客户端指针

#### 2.4 ready_keys 

ready_keys字典保存着阻塞键的等待队列，用于避免重复的阻塞命令

- 每个键都是一个指针，指向一个阻塞键
- 每个值都是NULL

#### 2.5 watched_keys

watched_keys字典保存着被客户端WATCH命令监控的键

- 每个键都是一个指针，指向被监控的键
- 每个值都是一个双向链表，存储着监控这个键的客户端

#### 2.6 读写键的维护 

- 读取一个键后，更新其LRU或者LFU，通过OBJECT idletime/freq 可以查询
- 读取一个键后，更新hits或者miss，通过INFO stats 可以查询统计
- 读取一个键时，发现其已经过期，会先删除这个键
- 修改一个键后，会对服务器dirty加1，这个计数器用于持久化和复制操作

### 3. Redis键过期策略

#### 3.1 设置键的生存时间或过期时间

通过EXPIRE或PEXPIRE命令可以设置键的过期，redis底层过期存储的是毫秒数。

通过TTL命令或PTTL命令可以查询键还有多久过期。

#### 3.2 过期键删除

- 惰性删除

  过期键的惰性删除由expireIfNeed函数实现，所以读写键值对的命令都会调用该函数检查键是否过期。在集群模式中，master节点会对过期键直接删除，但在slave节点中，只是逻辑上的删除，实际仍然存在，只有在master通知slave节点删除时，过期键才会真正的删除。这样由master节点统一管理过期键，在一定程度上保持了一致性。

- 定时删除

  定时过期删除由activeExpireCycle函数实现，当redis执行周期性操作时，该函数会被调用，在规定的时间内，分多次遍历多个数据库，从数据库expires中随机获取，检查过期并删除。

> 惰性删除中有dbAsyncDelete和dbSyncDelete两种策略，异步操作会将删除操作记录下来，积累到一定程度统一删除。

### 4. AOF、RDB和复制对过期键处理

- 生成RDB文件

  在执行SAVE或BGSAVE命令创建一个RDB文件时，会对数据库中过期的键进行检查，已过期的不会保存。

- 载入RDB文件

  启动服务器时，若开启RDB功能，那么会载入RDB文件，在集群模式中，master节点会对RDB中已经过期的键进行过滤，不加载，而slave节点则不会考虑该情况。

- AOF文件写入

  当过期键被删除时，回向AOF文件追加一条DEL指令，来表示键被删除

### 5. 源码剖析

- 检查键的过期

```c
int expireIfNeeded(redisDb *db, robj *key) {
    // 获取键的过期时间
    mstime_t when = getExpire(db,key);
    mstime_t now;

    // 没有过期时间返回0
    if (when < 0) return 0; /* No expire for this key */

    /* Don't expire anything while loading. It will be done later. */
    // 服务器加载时，不进行过期检查
    if (server.loading) return 0;

    /* If we are in the context of a Lua script, we pretend that time is
     * blocked to when the Lua script started. This way a key can expire
     * only the first time it is accessed and not in the middle of the
     * script execution, making propagation to slaves / AOF consistent.
     * See issue #1525 on Github for more information. */
    // 获取当前时间，lua脚本则是脚本执行开始时间
    now = server.lua_caller ? server.lua_time_start : mstime();

    /* If we are running in the context of a slave, return ASAP:
     * the slave key expiration is controlled by the master that will
     * send us synthesized DEL operations for expired keys.
     *
     * Still we try to return the right information to the caller,
     * that is, 0 if we think the key should be still valid, 1 if
     * we think the key is expired at this time. */
    // 如果是从节点，尽快返回已经过期，但并未删除
    // 从节点键过期需要通过master节点发送DEL操作执行
    if (server.masterhost != NULL) return now > when;

    /* Return when this key has not expired */
    // 键未过期，返回0
    if (now <= when) return 0;

    /* Delete the key */
    // 服务器过期键数+1
    server.stat_expiredkeys++;

    // 传递过期信息给从节点和AOF文件
    propagateExpire(db,key,server.lazyfree_lazy_expire);

    // "expired" 事件通知
    notifyKeyspaceEvent(NOTIFY_EXPIRED,
        "expired",key,db->id);

    // 懒过期则异步删除，否则同步删除
    return server.lazyfree_lazy_expire ? dbAsyncDelete(db,key) :
                                         dbSyncDelete(db,key);
}
```

- keys命令

```c
void keysCommand(client *c) {
    dictIterator *di;
    dictEntry *de;

    // 获取pattern
    sds pattern = c->argv[1]->ptr;
    int plen = sdslen(pattern), allkeys;
    unsigned long numkeys = 0;

    // 创建一个空链表，用于回复客户端
    void *replylen = addDeferredMultiBulkLength(c);

    // 字典安全迭代器
    di = dictGetSafeIterator(c->db->dict);
    // 所用的键
    allkeys = (pattern[0] == '*' && pattern[1] == '\0');

    // 遍历字典
    while((de = dictNext(di)) != NULL) {
        // 获取字典中的key
        sds key = dictGetKey(de);
        robj *keyobj;

        // 如果返回所有键或匹配正则
        if (allkeys || stringmatchlen(pattern,plen,key,sdslen(key),0)) {
            // 创建字符串对象
            keyobj = createStringObject(key,sdslen(key));

            // 检查键对象是否过期，没过期则回复客户端
            if (expireIfNeeded(c->db,keyobj) == 0) {
                addReplyBulk(c,keyobj);
                numkeys++;
            }
            decrRefCount(keyobj);
        }
    }
    // 释放迭代器
    dictReleaseIterator(di);
    // 设置回复客户端的长度
    setDeferredMultiBulkLength(c,replylen,numkeys);
}
```

- scan命令底层实现，利用字典的反向二进制扫描

```c
void scanGenericCommand(client *c, robj *o, unsigned long cursor) {
    int i, j;
    list *keys = listCreate();
    listNode *node, *nextnode;
    long count = 10;
    sds pat = NULL;
    int patlen = 0, use_pattern = 0;
    dict *ht;

    /* Object must be NULL (to iterate keys names), or the type of the object
     * must be Set, Sorted Set, or Hash. */
    // 必须是NULL(迭代key)，SET, ZSET 或者HASH
    serverAssert(o == NULL || o->type == OBJ_SET || o->type == OBJ_HASH ||
                o->type == OBJ_ZSET);

    /* Set i to the first option argument. The previous one is the cursor. */
    // 计算第一个选项参数的位置
    i = (o == NULL) ? 2 : 3; /* Skip the key argument if needed. */

    /* Step 1: Parse options. */
    // 第一步：解析选项
    while (i < c->argc) {
        j = c->argc - i;
        // COUNT选项，用于用户迭代每次返回多少元素
        if (!strcasecmp(c->argv[i]->ptr, "count") && j >= 2) {
            // 获取count参数
            if (getLongFromObjectOrReply(c, c->argv[i+1], &count, NULL)
                != C_OK)
            {
                goto cleanup;
            }

            // count不能小于1
            if (count < 1) {
                addReply(c,shared.syntaxerr);
                goto cleanup;
            }

            i += 2;

            // match选项：命令只返回匹配的元素
        } else if (!strcasecmp(c->argv[i]->ptr, "match") && j >= 2) {
            // pattern 字符串
            pat = c->argv[i+1]->ptr;
            patlen = sdslen(pat);

            /* The pattern always matches if it is exactly "*", so it is
             * equivalent to disabling it. */
            // 如果是* 表示所有，不用正则匹配
            use_pattern = !(pat[0] == '*' && patlen == 1);

            i += 2;
        } else {
            addReply(c,shared.syntaxerr);
            goto cleanup;
        }
    }

    /* Step 2: Iterate the collection.
     *
     * Note that if the object is encoded with a ziplist, intset, or any other
     * representation that is not a hash table, we are sure that it is also
     * composed of a small number of elements. So to avoid taking state we
     * just return everything inside the object in a single call, setting the
     * cursor to zero to signal the end of the iteration. */

    // 第二步：迭代集合
    // 注意：底层不是HASH，表示只含有少量元素，我们可以一次性全部返还，并将cursor设为0

    /* Handle the case of a hash table. */
    // 获取不同类型对象的哈希表
    ht = NULL;
    if (o == NULL) {
        ht = c->db->dict;
    } else if (o->type == OBJ_SET && o->encoding == OBJ_ENCODING_HT) {
        ht = o->ptr;
    } else if (o->type == OBJ_HASH && o->encoding == OBJ_ENCODING_HT) {
        ht = o->ptr;
        count *= 2; /* We return key / value for this type. */
    } else if (o->type == OBJ_ZSET && o->encoding == OBJ_ENCODING_SKIPLIST) {
        zset *zs = o->ptr;
        ht = zs->dict;
        count *= 2; /* We return key / value for this type. */
    }

    if (ht) {
        void *privdata[2];
        /* We set the max number of iterations to ten times the specified
         * COUNT, so if the hash table is in a pathological state (very
         * sparsely populated) we avoid to block too much time at the cost
         * of returning no or very few elements. */
        // 最大迭代长度
        long maxiterations = count*10;

        /* We pass two pointers to the callback: the list to which it will
         * add new elements, and the object containing the dictionary so that
         * it is possible to fetch more data in a type-dependent way. */
        // 回调函数scanCallback的参数
        privdata[0] = keys;
        privdata[1] = o;

        do {
            // 字典扫描，并返回cursor
            cursor = dictScan(ht, cursor, scanCallback, NULL, privdata);
        } while (cursor &&
              maxiterations-- &&
              listLength(keys) < (unsigned long)count);
    } else if (o->type == OBJ_SET) {
        int pos = 0;
        int64_t ll;

        // 整数集合全部获取返回
        while(intsetGet(o->ptr,pos++,&ll))
            listAddNodeTail(keys,createStringObjectFromLongLong(ll));
        cursor = 0;
    } else if (o->type == OBJ_HASH || o->type == OBJ_ZSET) {
        unsigned char *p = ziplistIndex(o->ptr,0);
        unsigned char *vstr;
        unsigned int vlen;
        long long vll;

        // 压缩列表遍历
        while(p) {
            // 获取压缩列表结点的值
            ziplistGet(p,&vstr,&vlen,&vll);
            listAddNodeTail(keys,
                (vstr != NULL) ? createStringObject((char*)vstr,vlen) :
                                 createStringObjectFromLongLong(vll));
            p = ziplistNext(o->ptr,p);
        }
        cursor = 0;
    } else {
        serverPanic("Not handled encoding in SCAN.");
    }

    /* Step 3: Filter elements. */
    // 第三步：过滤元素（设置了match参数）

    // 扫描key链表首地址
    node = listFirst(keys);
    while (node) {
        // 获取结点key
        robj *kobj = listNodeValue(node);
        // 下一结点地址
        nextnode = listNextNode(node);
        int filter = 0;

        /* Filter element if it does not match the pattern. */
        // 过滤元素
        if (!filter && use_pattern) {
            // kobj是字符串对象，则直接比较
            if (sdsEncodedObject(kobj)) {
                if (!stringmatchlen(pat, patlen, kobj->ptr, sdslen(kobj->ptr), 0))
                    filter = 1;
            } else {
                // kobj是正数对象，则转换后比较
                char buf[LONG_STR_SIZE];
                int len;

                serverAssert(kobj->encoding == OBJ_ENCODING_INT);
                len = ll2string(buf,sizeof(buf),(long)kobj->ptr);
                if (!stringmatchlen(pat, patlen, buf, len, 0)) filter = 1;
            }
        }

        /* Filter element if it is an expired key. */
        // 过滤过期的键
        if (!filter && o == NULL && expireIfNeeded(c->db, kobj)) filter = 1;

        /* Remove the element and its associted value if needed. */
        // 过滤则删除keys链表中的结点
        if (filter) {
            decrRefCount(kobj);
            listDelNode(keys, node);
        }

        /* If this is a hash or a sorted set, we have a flat list of
         * key-value elements, so if this element was filtered, remove the
         * value, or skip it if it was not filtered: we only match keys. */
        // 如果迭代模板是有序集合或哈希对象
        // keys保存的是键值对，需要将值对象也过滤掉
        if (o && (o->type == OBJ_ZSET || o->type == OBJ_HASH)) {
            node = nextnode;
            // 下一个结点地址
            nextnode = listNextNode(node);
            if (filter) {
                // 取出值对象
                kobj = listNodeValue(node);
                // 释放引用
                decrRefCount(kobj);
                // 删除keys中的结点
                listDelNode(keys, node);
            }
        }
        node = nextnode;
    }

    /* Step 4: Reply to the client. */
    // 第四步：回复客户端

    addReplyMultiBulkLen(c, 2);
    addReplyBulkLongLong(c,cursor);

    // 回复列表长度
    addReplyMultiBulkLen(c, listLength(keys));

    // 遍历回复列表，进行回复
    while ((node = listFirst(keys)) != NULL) {
        robj *kobj = listNodeValue(node);
        addReplyBulk(c, kobj);
        decrRefCount(kobj);
        listDelNode(keys, node);
    }

cleanup:
    listSetFreeMethod(keys,decrRefCountVoid);
    listRelease(keys);
}
```

- 过期信息传递

```c
/**
 * 传递过期信息给从节点和AOF文件
 * 当master结点的键过期时，DEL操作会发送给所有的从节点和AOF文件
 * 这种键过期的方式在一个地方集中管理，并且保证了操作的顺序
 */
void propagateExpire(redisDb *db, robj *key, int lazy) {
    robj *argv[2];

    // 构造参数列表
    argv[0] = lazy ? shared.unlink : shared.del;
    argv[1] = key;

    // 增加参数列表的引用计数
    incrRefCount(argv[0]);
    incrRefCount(argv[1]);

    // AOF状态开启，将del追加到AOF文件
    if (server.aof_state != AOF_OFF)
        feedAppendOnlyFile(server.delCommand,db->id,argv,2);

    // 将argv命令发送给所有从节点
    replicationFeedSlaves(server.slaves,db->id,argv,2);

    // 释放参数列表
    decrRefCount(argv[0]);
    decrRefCount(argv[1]);
}
```

