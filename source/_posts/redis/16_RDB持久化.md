---
title: Redis源码阅读(十六) RDB持久化
date: 2018-09-03 17:01:12
tags:
- redis
categories:
- 源码阅读
---

Redis是一个内存数据库，数据库的状态都存储在内存中，如果服务器进程退出，那么所有的服务器状态都会丢失。Redis提供了RDB持久化功能，可以将内存中数据库的状态保存到磁盘中，避免因为意外进程退出导致的数据库状态丢失。

###  1. RDB文件

- RDB存储对象长度定义，用来表示存储当前对象需要多少字节

| 宏定义       | 值         | 描述                                                 |
| ------------ | ---------- | :--------------------------------------------------- |
| RDB_6BITLEN  | 0          | 00XXXXXX，长度存储在后8bits中                        |
| RDB_14BITLEN | 1          | 01XXXXXX XXXXXXXX 长度存储在后14bits中               |
| RDB_32BITLEN | 0x80       | 10000000 [32 bit] 长度存储在后32bits中               |
| RDB_64BITLEN | 0x81       | 10000001 [64 bit] 长度存储在后64bits中               |
| RDB_ENCVAL   | 3          | 11OBKIND 表示一个特殊编码对象，用REDIS_RDS_ENC_*指定 |
| RDB_LENERR   | UINT64_MAX | 表示长度错误                                         |

- RDB对象编码类型，RDB_ENCVAL定义长度情况下， 用该编码类型表示

| 宏定义        | 值   | 描述           |
| ------------- | ---- | -------------- |
| RDB_ENC_INT8  | 0    | 8位有符号整数  |
| RDB_ENC_INT16 | 1    | 16位有符号整数 |
| RDB_ENC_INT32 | 2    | 32位有符号整数 |
| RDB_ENC_LZF   | 3    | LZF压缩字符串  |

- RDB对象存储类型，将Redis中的对象存储到RDB文件中保存的类型

| 宏定义            | 值   | 描述                                 |
| ----------------- | ---- | ------------------------------------ |
| RDB_TYPE_STRING   | 0    | 字符串对象                           |
| RDB_TYPE_LIST     | 1    | 列表对象                             |
| RDB_TYPE_SET      | 2    | 集合对象                             |
| RDB_TYPE_ZSET     | 3    | 有序集合对象                         |
| RDB_TYPE_HASH     | 4    | 哈希对象                             |
| RDB_TYPE_ZSET_2   | 5    | 有序集合对象版本2，score用二进制存储 |
| RDB_TYPE_MODULE   | 6    | 模块对象                             |
| RDB_TYPE_MODULE_2 | 7    | 带有注释的模块对象，用于解析而不生成 |

- RDB对象存储类型的编码方式

| 宏定义                  | 值   | 描述                             |
| ----------------------- | ---- | -------------------------------- |
| RDB_TYPE_HASH_ZIPMAP    | 9    | ZIPMAP编码的哈希对象，已不再使用 |
| RDB_TYPE_LIST_ZIPLIST   | 10   | ZIPLIST编码的列表对象            |
| RDB_TYPE_SET_INTSET     | 11   | INTSET编码的集合对象             |
| RDB_TYPE_ZSET_ZIPLIST   | 12   | ZIPLIST编码的有序集合对象        |
| RDB_TYPE_HASH_ZIPLIST   | 13   | ZIPLIST编码的哈希对象            |
| RDB_TYPE_LIST_QUICKLIST | 14   | QUICKLIST编码的列表对象          |

- RDB操作码，保存和加载类型时使用

| 宏定义                   | 值   | 描述           |
| ------------------------ | ---- | -------------- |
| RDB_OPCODE_AUX           | 250  | AUX操作        |
| RDB_OPCODE_RESIZEDB      | 251  | 需要resize操作 |
| RDB_OPCODE_EXPIRETIME_MS | 252  | 过期毫秒数     |
| RDB_OPCODE_EXPIRETIME    | 253  | 过期秒数       |
| RDB_OPCODE_SELECTDB      | 254  | 选择db操作     |
| RDB_OPCODE_EOF           | 255  | 结束操作       |

#### 1.1 RDB文件结构

| REDIS   | VERSION | DATABASE | EOF      | CHECKNUM |
| ------- | ------- | -------- | -------- | -------- |
| "REDIS" | 版本号  | 数据区域 | 结束标识 | 校验和   |

RDB文件以“REDIS”开头，后紧接着4字节的版本号。

database数据区域存储着一个或多个特定格式编码的数据库。

EOF是RDB正文结束。

check_num是校验和，通过前4部分计算得出。

#### 1.2 database数据区域部分

一个RDB文件可能存在多个非空数据库，通过OPCODE_SELECTDB操作码可以选择数据库对象，格式

| RDB_OPCODE_SELECTDB  | db_number  | key_value_pairs  |
| -------------------- | ---------- | ---------------- |
| 选择数据库操作码 254 | 数据库编号 | 所有的键值对数据 |

#### 1.3 键值对数据区域

RDB文件中每个键值对数据区域都保存着一个或多个键值对，如果带有过期时间，那么过期时间也会被保存。

| RDB_OPCODE_EXPIRETIME | ms       | TYPE   | key  | value |
| --------------------- | -------- | ------ | ---- | ----- |
| 过期时间操作码        | 过期时间 | 键类型 | 键   | 值    |

### 2. RDB文件的创建

SAVE命令和BGSAVE命令可以创建RDB文件。由于Redis是单进程的，所以SAVE命令创建RDB文件会阻塞Redis服务器，这种情况下，服务器不能处理任何命令请求。BGSAVE则会生成一个子进程用于创建RDB文件，父进程则进行处理命令请求。

> 注：在同一时间内，只能存在一个BGSAVE子进程。

RDB文件创建由rdb.c/rdbSave函数实现：

1. 创建临时文件temp-pid.rdb，用于保存RDB内容

2. 将数据库内容按照一定编码格式写入文件

   1）计算校验和

   2）存储redis版本

   3）遍历服务器数据库并写入到文件，内存内容到RDB需要按照一定的编码格式和规则

   4）存在lua脚本也需要备份脚本

   5）存储EOF操作码

   6）存储校验和

3. 修改临时文件名为RDB文件名

4. 记录服务端操作信息

BGSAVE命令同样会调用rdbSave函数，区别是在fork出的子进程中执行该函数。

### 3. RDB文件的加载

在Redis服务器启动时，会根据配置的选项进行初始化，若开启了RDB功能，且未开启AOF功能，则会从RDB文件中加载数据到内存中。加载过程中，Redis也是处于阻塞状态，不能处理其他服务指令。

> AOF文件更新频率通常高于RDB，所以如果开启了AOF功能，服务器会首先使用AOF文件还原数据库状态。

RDB文件的加载由rdb.c/rdbLoad函数实现：

1. 打开RDB文件

2. 开始加载

3. 初始化Redis输入输出缓存对象

4. 读取RDB文件中的数据到内存

   1）设置计算校验和函数

   2）设置加载读或写的最大字节数

   3）读取9字节，这9字节存储着redis的版本信息，并判断版本是否正确

   4）循环读取RDB文件，按照写入RDB文件的编码方式进行读取

   5）Redis版本大于5则需要进行校验计算

5. 关闭文件

6. 加载完成



### 4. 源码剖析

- RDB创建实现

```c
int rdbSaveRio(rio *rdb, int *error, int flags, rdbSaveInfo *rsi) {
    dictIterator *di = NULL;
    dictEntry *de;
    char magic[10];
    int j;
    uint64_t cksum;
    size_t processed = 0;

    // 校验和选项
    if (server.rdb_checksum)
        rdb->update_cksum = rioGenericUpdateChecksum;

    // 存储redis版本到magic
    snprintf(magic,sizeof(magic),"REDIS%04d",RDB_VERSION);

    // mgaic写入到RDB
    if (rdbWriteRaw(rdb,magic,9) == -1) goto werr;

    // 将RDB文件的默认信息写入
    if (rdbSaveInfoAuxFields(rdb,flags,rsi) == -1) goto werr;

    // 遍历服务端数据库
    for (j = 0; j < server.dbnum; j++) {
        // 当前数据库指针
        redisDb *db = server.db+j;
        // 数据库的键值对字典
        dict *d = db->dict;
        // 字典大小为0下个数据库
        if (dictSize(d) == 0) continue;

        // 获取字典安全迭代器
        di = dictGetSafeIterator(d);
        if (!di) return C_ERR;

        /* Write the SELECT DB opcode */
        // 写入数据库的选择标识码RDB_OPCODE_SELECTDB为254
        if (rdbSaveType(rdb,RDB_OPCODE_SELECTDB) == -1) goto werr;
        // 写入数据库id，占用一字节长度
        if (rdbSaveLen(rdb,j) == -1) goto werr;

        /* Write the RESIZE DB opcode. We trim the size to UINT32_MAX, which
         * is currently the largest type we are able to represent in RDB sizes.
         * However this does not limit the actual size of the DB to load since
         * these sizes are just hints to resize the hash tables. */
        uint32_t db_size, expires_size;
        // 我们将大小限制在UNIT32_MAX，但不代表实际大小，只是用于提示重新调整哈希表大小
        // 计算键值对字典大小
        db_size = (dictSize(db->dict) <= UINT32_MAX) ?
                                dictSize(db->dict) :
                                UINT32_MAX;
        // 计算过期字典大小
        expires_size = (dictSize(db->expires) <= UINT32_MAX) ?
                                dictSize(db->expires) :
                                UINT32_MAX;
        // 写入哈希表调整嘛RDB_OPCODE_RESIZEDB=251
        if (rdbSaveType(rdb,RDB_OPCODE_RESIZEDB) == -1) goto werr;
        // 写入键值对字典大小
        if (rdbSaveLen(rdb,db_size) == -1) goto werr;
        // 写入过期字典大小
        if (rdbSaveLen(rdb,expires_size) == -1) goto werr;

        /* Iterate this DB writing every entry */
        // 遍历键值对字典
        while((de = dictNext(di)) != NULL) {
            // 键
            sds keystr = dictGetKey(de);
            // 值
            robj key, *o = dictGetVal(de);
            long long expire;

            // 栈空间中创建一个键对象
            initStaticStringObject(key,keystr);
            // 获取当前键的过期时间
            expire = getExpire(db,&key);
            // 保存键值对和过期时间到RDB
            if (rdbSaveKeyValuePair(rdb,&key,o,expire) == -1) goto werr;

            /* When this RDB is produced as part of an AOF rewrite, move
             * accumulated diff from parent to child while rewriting in
             * order to have a smaller final write. */
            // 当这个RDB作为AOF重写的一部分生成时，
            // 在重写时移动父进程到子进程的累积差异,
            // 是为了获取更小的写入大小
            if (flags & RDB_SAVE_AOF_PREAMBLE &&
                rdb->processed_bytes > processed+AOF_READ_DIFF_INTERVAL_BYTES)
            {
                processed = rdb->processed_bytes;
                aofReadDiffFromParent();
            }
        }
        // 释放迭代器
        dictReleaseIterator(di);
    }
    di = NULL; /* So that we don't release it again on error. */

    /* If we are storing the replication information on disk, persist
     * the script cache as well: on successful PSYNC after a restart, we need
     * to be able to process any EVALSHA inside the replication backlog the
     * master will send us. */
    // 备份服务器的lua脚本
    if (rsi && dictSize(server.lua_scripts)) {
        di = dictGetIterator(server.lua_scripts);
        while((de = dictNext(di)) != NULL) {
            robj *body = dictGetVal(de);
            if (rdbSaveAuxField(rdb,"lua",3,body->ptr,sdslen(body->ptr)) == -1)
                goto werr;
        }
        dictReleaseIterator(di);
    }

    /* EOF opcode */
    // 结束标识符RDB_OPCODE_EOF=255
    if (rdbSaveType(rdb,RDB_OPCODE_EOF) == -1) goto werr;

    /* CRC64 checksum. It will be zero if checksum computation is disabled, the
     * loading code skips the check in this case. */
    // CRC64校验和. 校验和计算未开启时为0，加载RDB文件时会跳过
    cksum = rdb->cksum;
    memrev64ifbe(&cksum);
    if (rioWrite(rdb,&cksum,8) == 0) goto werr;
    return C_OK;

werr:
    if (error) *error = errno;
    if (di) dictReleaseIterator(di);
    return C_ERR;
}
```

- RDB读取实现

```c
int rdbLoadRio(rio *rdb, rdbSaveInfo *rsi, int loading_aof) {
    uint64_t dbid;
    int type, rdbver;
    redisDb *db = server.db+0;
    char buf[1024];
    long long expiretime, now = mstime();

    // 设置计算校验和函数
    rdb->update_cksum = rdbLoadProgressCallback;
    // 设置加载读或写的最大字节数
    rdb->max_processing_chunk = server.loading_process_events_interval_bytes;
    // 读取9字节，buf中保存redis版本信息"redis0001"
    if (rioRead(rdb,buf,9) == 0) goto eoferr;
    buf[9] = '\0';
    if (memcmp(buf,"REDIS",5) != 0) {
        serverLog(LL_WARNING,"Wrong signature trying to load DB from file");
        errno = EINVAL;
        return C_ERR;
    }

    // redis版本
    rdbver = atoi(buf+5);
    if (rdbver < 1 || rdbver > RDB_VERSION) {
        serverLog(LL_WARNING,"Can't handle RDB format version %d",rdbver);
        errno = EINVAL;
        return C_ERR;
    }

    // 读取RDB文件
    while(1) {
        robj *key, *val;
        expiretime = -1;

        /* Read type. */
        // 读取类型
        if ((type = rdbLoadType(rdb)) == -1) goto eoferr;

        /* Handle special types. */
        // 处理特殊类型

        // 过期时间类型
        if (type == RDB_OPCODE_EXPIRETIME) {
            /* EXPIRETIME: load an expire associated with the next key
             * to load. Note that after loading an expire we need to
             * load the actual type, and continue. */
            // 读取过期时间
            if ((expiretime = rdbLoadTime(rdb)) == -1) goto eoferr;
            /* We read the time so we need to read the object type again. */
            // 读取下一个键值对类型
            if ((type = rdbLoadType(rdb)) == -1) goto eoferr;
            /* the EXPIRETIME opcode specifies time in seconds, so convert
             * into milliseconds. */
            // 转换为毫秒
            expiretime *= 1000;

            // 过期时间毫秒类型
        } else if (type == RDB_OPCODE_EXPIRETIME_MS) {
            /* EXPIRETIME_MS: milliseconds precision expire times introduced
             * with RDB v3. Like EXPIRETIME but no with more precision. */
            // 读取过期时间
            if ((expiretime = rdbLoadMillisecondTime(rdb)) == -1) goto eoferr;
            /* We read the time so we need to read the object type again. */
            // 读取下一个键值对类型
            if ((type = rdbLoadType(rdb)) == -1) goto eoferr;

            // 终止EOF, 读取到则终止循环，表示加载完成
        } else if (type == RDB_OPCODE_EOF) {
            /* EOF: End of file, exit the main loop. */
            break;

            // 读取切换数据库操作
        } else if (type == RDB_OPCODE_SELECTDB) {
            /* SELECTDB: Select the specified database. */
            // 读取数据库id
            if ((dbid = rdbLoadLen(rdb,NULL)) == RDB_LENERR)
                goto eoferr;
            // 检查数据库id的合法性
            if (dbid >= (unsigned)server.dbnum) {
                serverLog(LL_WARNING,
                    "FATAL: Data file was created with a Redis "
                    "server configured to handle more than %d "
                    "databases. Exiting\n", server.dbnum);
                exit(1);
            }
            // 更换数据库
            db = server.db+dbid;
            continue; /* Read type again. */

            // 读取RESIZEDB操作，重新调整哈希表大小
        } else if (type == RDB_OPCODE_RESIZEDB) {
            /* RESIZEDB: Hint about the size of the keys in the currently
             * selected data base, in order to avoid useless rehashing. */
            uint64_t db_size, expires_size;
            // 读取键值对字典的大小
            if ((db_size = rdbLoadLen(rdb,NULL)) == RDB_LENERR)
                goto eoferr;
            // 读取过期字典的大小
            if ((expires_size = rdbLoadLen(rdb,NULL)) == RDB_LENERR)
                goto eoferr;
            // 字典扩展
            dictExpand(db->dict,db_size);
            dictExpand(db->expires,expires_size);
            continue; /* Read type again. */

            // 读取AUX，辅助信息
        } else if (type == RDB_OPCODE_AUX) {
            /* AUX: generic string-string fields. Use to add state to RDB
             * which is backward compatible. Implementations of RDB loading
             * are requierd to skip AUX fields they don't understand.
             *
             * An AUX field is composed of two strings: key and value. */
            robj *auxkey, *auxval;
            // 读取辅助字典
            if ((auxkey = rdbLoadStringObject(rdb)) == NULL) goto eoferr;
            if ((auxval = rdbLoadStringObject(rdb)) == NULL) goto eoferr;

            if (((char*)auxkey->ptr)[0] == '%') {
                /* All the fields with a name staring with '%' are considered
                 * information fields and are logged at startup with a log
                 * level of NOTICE. */
                serverLog(LL_NOTICE,"RDB '%s': %s",
                    (char*)auxkey->ptr,
                    (char*)auxval->ptr);
            } else if (!strcasecmp(auxkey->ptr,"repl-stream-db")) {
                if (rsi) rsi->repl_stream_db = atoi(auxval->ptr);
            } else if (!strcasecmp(auxkey->ptr,"repl-id")) {
                if (rsi && sdslen(auxval->ptr) == CONFIG_RUN_ID_SIZE) {
                    memcpy(rsi->repl_id,auxval->ptr,CONFIG_RUN_ID_SIZE+1);
                    rsi->repl_id_is_set = 1;
                }
            } else if (!strcasecmp(auxkey->ptr,"repl-offset")) {
                if (rsi) rsi->repl_offset = strtoll(auxval->ptr,NULL,10);
            } else if (!strcasecmp(auxkey->ptr,"lua")) {
                /* Load the script back in memory. */
                if (luaCreateFunction(NULL,server.lua,auxval) == NULL) {
                    rdbExitReportCorruptRDB(
                        "Can't load Lua script from RDB file! "
                        "BODY: %s", auxval->ptr);
                }
            } else {
                /* We ignore fields we don't understand, as by AUX field
                 * contract. */
                serverLog(LL_DEBUG,"Unrecognized RDB AUX field: '%s'",
                    (char*)auxkey->ptr);
            }

            decrRefCount(auxkey);
            decrRefCount(auxval);
            continue; /* Read type again. */
        }

        /* Read key */
        // 读取键对象
        if ((key = rdbLoadStringObject(rdb)) == NULL) goto eoferr;
        /* Read value */
        // 读取值对象
        if ((val = rdbLoadObject(type,rdb)) == NULL) goto eoferr;
        /* Check if the key already expired. This function is used when loading
         * an RDB file from disk, either at startup, or when an RDB was
         * received from the master. In the latter case, the master is
         * responsible for key expiry. If we would expire keys here, the
         * snapshot taken by the master may not be reflected on the slave. */
        // 如果是master节点，且已经过期，则释放该键值对
        if (server.masterhost == NULL && !loading_aof && expiretime != -1 && expiretime < now) {
            decrRefCount(key);
            decrRefCount(val);
            continue;
        }
        /* Add the new object in the hash table */
        // 添加键值对数据库
        dbAdd(db,key,val);

        /* Set the expire time if needed */
        // 设置过期时间
        if (expiretime != -1) setExpire(NULL,db,key,expiretime);

        // 释放临时对象key
        decrRefCount(key);
    }
    /* Verify the checksum if RDB version is >= 5 */
    // redis版本大于5，需要进行校验
    if (rdbver >= 5) {
        uint64_t cksum, expected = rdb->cksum;

        if (rioRead(rdb,&cksum,8) == 0) goto eoferr;
        if (server.rdb_checksum) {
            memrev64ifbe(&cksum);
            if (cksum == 0) {
                serverLog(LL_WARNING,"RDB file was saved with checksum disabled: no check performed.");
            } else if (cksum != expected) {
                serverLog(LL_WARNING,"Wrong RDB checksum. Aborting now.");
                rdbExitReportCorruptRDB("RDB CRC error");
            }
        }
    }
    return C_OK;

eoferr: /* unexpected end of file is handled here with a fatal exit */
    serverLog(LL_WARNING,"Short read or OOM loading DB. Unrecoverable error, aborting now.");
    rdbExitReportCorruptRDB("Unexpected EOF reading RDB file");
    return C_ERR; /* Just to avoid warning */
}
```

- RDB创建一个对象

```c
ssize_t rdbSaveObject(rio *rdb, robj *o) {
    ssize_t n = 0, nwritten = 0;

    // 字符串对象
    if (o->type == OBJ_STRING) {
        /* Save a string value */
        // 保存字符串对象
        if ((n = rdbSaveStringObject(rdb,o)) == -1) return -1;
        nwritten += n;

        // 列表对象
    } else if (o->type == OBJ_LIST) {
        /* Save a list value */
        // quicklist编码
        if (o->encoding == OBJ_ENCODING_QUICKLIST) {
            quicklist *ql = o->ptr;
            quicklistNode *node = ql->head;

            // 先保存长度
            if ((n = rdbSaveLen(rdb,ql->len)) == -1) return -1;
            nwritten += n;

            // 然后遍历quicklist中的结点
            while(node) {
                // 根据是否压缩分别存储
                if (quicklistNodeIsCompressed(node)) {
                    void *data;
                    size_t compress_len = quicklistGetLzf(node, &data);
                    // 压缩过，则解压后保存二进制数据到rdb
                    if ((n = rdbSaveLzfBlob(rdb,data,compress_len,node->sz)) == -1) return -1;
                    nwritten += n;
                } else {
                    // 未压缩过，则直接保存原始的字符串数据
                    if ((n = rdbSaveRawString(rdb,node->zl,node->sz)) == -1) return -1;
                    nwritten += n;
                }
                // 下一个结点
                node = node->next;
            }
        } else {
            serverPanic("Unknown list encoding");
        }

        // 集合对象
    } else if (o->type == OBJ_SET) {
        /* Save a set value */
        // 哈希表编码
        if (o->encoding == OBJ_ENCODING_HT) {
            dict *set = o->ptr;
            // 获取集合对象迭代器
            dictIterator *di = dictGetIterator(set);
            dictEntry *de;

            // 保存字典长度到RDB
            if ((n = rdbSaveLen(rdb,dictSize(set))) == -1) return -1;
            nwritten += n;

            // 遍历字典
            while((de = dictNext(di)) != NULL) {
                // 获取sds字符串对象
                sds ele = dictGetKey(de);
                // 保存到RDB中
                if ((n = rdbSaveRawString(rdb,(unsigned char*)ele,sdslen(ele)))
                    == -1) return -1;
                nwritten += n;
            }

            // 释放迭代器
            dictReleaseIterator(di);

            // 整数集合编码
        } else if (o->encoding == OBJ_ENCODING_INTSET) {
            // 获取整数集合长度
            size_t l = intsetBlobLen((intset*)o->ptr);

            // 保存原始字符串到RDB
            if ((n = rdbSaveRawString(rdb,o->ptr,l)) == -1) return -1;
            nwritten += n;
        } else {
            serverPanic("Unknown set encoding");
        }

        // 有序集合对象
    } else if (o->type == OBJ_ZSET) {
        /* Save a sorted set value */
        // 压缩列表编码
        if (o->encoding == OBJ_ENCODING_ZIPLIST) {
            // 压缩列表长度
            size_t l = ziplistBlobLen((unsigned char*)o->ptr);

            // 存储到rdb
            if ((n = rdbSaveRawString(rdb,o->ptr,l)) == -1) return -1;
            nwritten += n;


            // 跳跃表编码
        } else if (o->encoding == OBJ_ENCODING_SKIPLIST) {
            zset *zs = o->ptr;
            zskiplist *zsl = zs->zsl;

            // 存储跳跃表长度到RDB
            if ((n = rdbSaveLen(rdb,zsl->length)) == -1) return -1;
            nwritten += n;

            /* We save the skiplist elements from the greatest to the smallest
             * (that's trivial since the elements are already ordered in the
             * skiplist): this improves the load process, since the next loaded
             * element will always be the smaller, so adding to the skiplist
             * will always immediately stop at the head, making the insertion
             * O(1) instead of O(log(N)). */
            zskiplistNode *zn = zsl->tail;
            // 遍历跳跃表
            while (zn != NULL) {
                // 成员ele存储原始字符串到RDB
                if ((n = rdbSaveRawString(rdb,
                    (unsigned char*)zn->ele,sdslen(zn->ele))) == -1)
                {
                    return -1;
                }
                nwritten += n;
                // 分数score存储二进制double到rdb
                if ((n = rdbSaveBinaryDoubleValue(rdb,zn->score)) == -1)
                    return -1;
                nwritten += n;
                zn = zn->backward;
            }
        } else {
            serverPanic("Unknown sorted set encoding");
        }

        // 哈希对象
    } else if (o->type == OBJ_HASH) {
        /* Save a hash value */
        // 压缩列表编码
        if (o->encoding == OBJ_ENCODING_ZIPLIST) {
            // 长度
            size_t l = ziplistBlobLen((unsigned char*)o->ptr);

            // 存储到RDB
            if ((n = rdbSaveRawString(rdb,o->ptr,l)) == -1) return -1;
            nwritten += n;

            // 哈希表编码
        } else if (o->encoding == OBJ_ENCODING_HT) {
            // 迭代器
            dictIterator *di = dictGetIterator(o->ptr);
            dictEntry *de;

            // 保存字典长度到RDB
            if ((n = rdbSaveLen(rdb,dictSize((dict*)o->ptr))) == -1) return -1;
            nwritten += n;

            // 遍历字典
            while((de = dictNext(di)) != NULL) {
                // 获取成员field
                sds field = dictGetKey(de);
                // 获取值value
                sds value = dictGetVal(de);

                // 存储field
                if ((n = rdbSaveRawString(rdb,(unsigned char*)field,
                        sdslen(field))) == -1) return -1;
                nwritten += n;
                // 存储value
                if ((n = rdbSaveRawString(rdb,(unsigned char*)value,
                        sdslen(value))) == -1) return -1;
                nwritten += n;
            }
            dictReleaseIterator(di);
        } else {
            serverPanic("Unknown hash encoding");
        }

        // 模块对象
    } else if (o->type == OBJ_MODULE) {
        /* Save a module-specific value. */
        RedisModuleIO io;
        moduleValue *mv = o->ptr;
        moduleType *mt = mv->type;
        moduleInitIOContext(io,mt,rdb);

        /* Write the "module" identifier as prefix, so that we'll be able
         * to call the right module during loading. */
        int retval = rdbSaveLen(rdb,mt->id);
        if (retval == -1) return -1;
        io.bytes += retval;

        /* Then write the module-specific representation + EOF marker. */
        mt->rdb_save(&io,mv->value);
        retval = rdbSaveLen(rdb,RDB_MODULE_OPCODE_EOF);
        if (retval == -1) return -1;
        io.bytes += retval;

        if (io.ctx) {
            moduleFreeContext(io.ctx);
            zfree(io.ctx);
        }
        return io.error ? -1 : (ssize_t)io.bytes;
    } else {
        serverPanic("Unknown object type");
    }
    return nwritten;
}
```

- RDB读取一个读写

```c
robj *rdbLoadObject(int rdbtype, rio *rdb) {
    robj *o = NULL, *ele, *dec;
    uint64_t len;
    unsigned int i;

    // 字符串对象
    if (rdbtype == RDB_TYPE_STRING) {
        /* Read string value */
        // 读取编码的字符串对象
        if ((o = rdbLoadEncodedStringObject(rdb)) == NULL) return NULL;
        // 优化编码
        o = tryObjectEncoding(o);

        // 列表对象
    } else if (rdbtype == RDB_TYPE_LIST) {
        /* Read list value */
        // 读取列表对象元素个数
        if ((len = rdbLoadLen(rdb,NULL)) == RDB_LENERR) return NULL;

        // 创建快速列表
        o = createQuicklistObject();
        // 设置压缩程度和ziplist最大长度
        quicklistSetOptions(o->ptr, server.list_max_ziplist_size,
                            server.list_compress_depth);

        /* Load every single element of the list */
        // 读取len个元素
        while(len--) {
            // 读取编码的字符串对象
            if ((ele = rdbLoadEncodedStringObject(rdb)) == NULL) return NULL;
            // 解码字符串对象
            dec = getDecodedObject(ele);
            // 获取字符串的长度
            size_t len = sdslen(dec->ptr);
            // 添加读取的字符串对象到quicklist
            quicklistPushTail(o->ptr, dec->ptr, len);
            // 释放临时变量
            decrRefCount(dec);
            decrRefCount(ele);
        }

        // 集合对象
    } else if (rdbtype == RDB_TYPE_SET) {
        /* Read Set value */
        // 读取集合对象中元素的个数
        if ((len = rdbLoadLen(rdb,NULL)) == RDB_LENERR) return NULL;

        /* Use a regular set when there are too many entries. */
        // 如果超过最多intset节点数，则创建一个HT编码的集合对象
        if (len > server.set_max_intset_entries) {
            o = createSetObject();
            /* It's faster to expand the dict to the right size asap in order
             * to avoid rehashing */
            if (len > DICT_HT_INITIAL_SIZE)
                dictExpand(o->ptr,len);
        } else {
            o = createIntsetObject();
        }

        /* Load every single element of the set */
        // 遍历len个元素
        for (i = 0; i < len; i++) {
            long long llval;
            sds sdsele;

            // 读取sds字符串
            if ((sdsele = rdbGenericLoadStringObject(rdb,RDB_LOAD_SDS,NULL))
                == NULL) return NULL;

            // INTSET编码
            if (o->encoding == OBJ_ENCODING_INTSET) {
                /* Fetch integer value from element. */
                // 读取整数值，并添加到INTSET
                if (isSdsRepresentableAsLongLong(sdsele,&llval) == C_OK) {
                    o->ptr = intsetAdd(o->ptr,llval,NULL);
                } else {
                    // 转换成哈希表
                    setTypeConvert(o,OBJ_ENCODING_HT);
                    dictExpand(o->ptr,len);
                }
            }

            /* This will also be called when the set was just converted
             * to a regular hash table encoded set. */
            // HT编码则添加到字典中
            if (o->encoding == OBJ_ENCODING_HT) {
                dictAdd((dict*)o->ptr,sdsele,NULL);
            } else {
                sdsfree(sdsele);
            }
        }

        // 有序集合对象
    } else if (rdbtype == RDB_TYPE_ZSET_2 || rdbtype == RDB_TYPE_ZSET) {
        /* Read list/set value. */
        uint64_t zsetlen;
        size_t maxelelen = 0;
        zset *zs;

        // 读取有序集合对象的元素个数
        if ((zsetlen = rdbLoadLen(rdb,NULL)) == RDB_LENERR) return NULL;
        // 创建有序集合对象
        o = createZsetObject();
        zs = o->ptr;

        /* Load every single element of the sorted set. */
        // 遍历len个元素
        while(zsetlen--) {
            sds sdsele;
            double score;
            zskiplistNode *znode;

            // 读取元素的key
            if ((sdsele = rdbGenericLoadStringObject(rdb,RDB_LOAD_SDS,NULL))
                == NULL) return NULL;

            // 读取元素的score
            if (rdbtype == RDB_TYPE_ZSET_2) {
                if (rdbLoadBinaryDoubleValue(rdb,&score) == -1) return NULL;
            } else {
                if (rdbLoadDoubleValue(rdb,&score) == -1) return NULL;
            }

            /* Don't care about integer-encoded strings. */
            // 保存成员最大长度
            if (sdslen(sdsele) > maxelelen) maxelelen = sdslen(sdsele);

            // 当前元素对象插入到跳跃表和字典中
            znode = zslInsert(zs->zsl,score,sdsele);
            dictAdd(zs->dict,sdsele,&znode->score);
        }

        /* Convert *after* loading, since sorted sets are not stored ordered. */
        // 尝试编码转换为ziplist，以节省内存空间
        if (zsetLength(o) <= server.zset_max_ziplist_entries &&
            maxelelen <= server.zset_max_ziplist_value)
                zsetConvert(o,OBJ_ENCODING_ZIPLIST);

        // 哈希对象
    } else if (rdbtype == RDB_TYPE_HASH) {
        uint64_t len;
        int ret;
        sds field, value;

        // 读取哈希对象元素个数
        len = rdbLoadLen(rdb, NULL);
        if (len == RDB_LENERR) return NULL;

        // 创建redis哈希对象
        o = createHashObject();

        /* Too many entries? Use a hash table. */
        // 如果超过ziplist的最大限制，转换为HT编码
        if (len > server.hash_max_ziplist_entries)
            hashTypeConvert(o, OBJ_ENCODING_HT);

        /* Load every field and value into the ziplist */
        // 读取元素的field和score存储到ziplist
        while (o->encoding == OBJ_ENCODING_ZIPLIST && len > 0) {
            len--;
            /* Load raw strings */
            // 读取field
            if ((field = rdbGenericLoadStringObject(rdb,RDB_LOAD_SDS,NULL))
                == NULL) return NULL;
            // 读取score
            if ((value = rdbGenericLoadStringObject(rdb,RDB_LOAD_SDS,NULL))
                == NULL) return NULL;

            /* Add pair to ziplist */
            // 存储field和score到ziplist
            o->ptr = ziplistPush(o->ptr, (unsigned char*)field,
                    sdslen(field), ZIPLIST_TAIL);
            o->ptr = ziplistPush(o->ptr, (unsigned char*)value,
                    sdslen(value), ZIPLIST_TAIL);

            /* Convert to hash table if size threshold is exceeded */
            // 如果超过的ziplist的限制，转换为HT编码
            if (sdslen(field) > server.hash_max_ziplist_value ||
                sdslen(value) > server.hash_max_ziplist_value)
            {
                sdsfree(field);
                sdsfree(value);
                hashTypeConvert(o, OBJ_ENCODING_HT);
                break;
            }

            // 释放field,value
            sdsfree(field);
            sdsfree(value);
        }

        /* Load remaining fields and values into the hash table */
        // HT编码，读取field和value存储到哈希表
        while (o->encoding == OBJ_ENCODING_HT && len > 0) {
            len--;
            /* Load encoded strings */
            // 读取field和score
            if ((field = rdbGenericLoadStringObject(rdb,RDB_LOAD_SDS,NULL))
                == NULL) return NULL;
            if ((value = rdbGenericLoadStringObject(rdb,RDB_LOAD_SDS,NULL))
                == NULL) return NULL;

            /* Add pair to hash table */
            // 存储到字典中
            ret = dictAdd((dict*)o->ptr, field, value);
            if (ret == DICT_ERR) {
                rdbExitReportCorruptRDB("Duplicate keys detected");
            }
        }

        /* All pairs should be read by now */
        serverAssert(len == 0);

        // QUICKLIST类型的列表
    } else if (rdbtype == RDB_TYPE_LIST_QUICKLIST) {
        // 读取quicklist结点个数
        if ((len = rdbLoadLen(rdb,NULL)) == RDB_LENERR) return NULL;

        // 创建一个quicklist编码的列表对象
        o = createQuicklistObject();
        // 设置压缩程度和ziplist的最大长度
        quicklistSetOptions(o->ptr, server.list_max_ziplist_size,
                            server.list_compress_depth);

        // 读取len个元素结点
        while (len--) {
            // 读取一个quicklistNode结点的ziplist地址
            unsigned char *zl =
                rdbGenericLoadStringObject(rdb,RDB_LOAD_PLAIN,NULL);
            if (zl == NULL) return NULL;
            // 添加ziplist到快速列表结点，然后将快速列表结点添加到quicklist中
            quicklistAppendZiplist(o->ptr, zl);
        }

        // 其他一些类型编码
    } else if (rdbtype == RDB_TYPE_HASH_ZIPMAP  ||
               rdbtype == RDB_TYPE_LIST_ZIPLIST ||
               rdbtype == RDB_TYPE_SET_INTSET   ||
               rdbtype == RDB_TYPE_ZSET_ZIPLIST ||
               rdbtype == RDB_TYPE_HASH_ZIPLIST)
    {
        // 读取字符串值
        unsigned char *encoded =
            rdbGenericLoadStringObject(rdb,RDB_LOAD_PLAIN,NULL);
        if (encoded == NULL) return NULL;

        // 创建字符串对象
        o = createObject(OBJ_STRING,encoded); /* Obj type fixed below. */

        /* Fix the object encoding, and make sure to convert the encoded
         * data type into the base type if accordingly to the current
         * configuration there are too many elements in the encoded data
         * type. Note that we only check the length and not max element
         * size as this is an O(N) scan. Eventually everything will get
         * converted. */
        // 更加读取编码类型，读取其value，并转化为原来的编码对象
        switch(rdbtype) {
            case RDB_TYPE_HASH_ZIPMAP: // 压缩字典，已经不再使用
                /* Convert to ziplist encoded hash. This must be deprecated
                 * when loading dumps created by Redis 2.4 gets deprecated. */
                {
                    unsigned char *zl = ziplistNew();
                    unsigned char *zi = zipmapRewind(o->ptr);
                    unsigned char *fstr, *vstr;
                    unsigned int flen, vlen;
                    unsigned int maxlen = 0;

                    while ((zi = zipmapNext(zi, &fstr, &flen, &vstr, &vlen)) != NULL) {
                        if (flen > maxlen) maxlen = flen;
                        if (vlen > maxlen) maxlen = vlen;
                        zl = ziplistPush(zl, fstr, flen, ZIPLIST_TAIL);
                        zl = ziplistPush(zl, vstr, vlen, ZIPLIST_TAIL);
                    }

                    zfree(o->ptr);
                    o->ptr = zl;
                    o->type = OBJ_HASH;
                    o->encoding = OBJ_ENCODING_ZIPLIST;

                    if (hashTypeLength(o) > server.hash_max_ziplist_entries ||
                        maxlen > server.hash_max_ziplist_value)
                    {
                        hashTypeConvert(o, OBJ_ENCODING_HT);
                    }
                }
                break;
            case RDB_TYPE_LIST_ZIPLIST: // ZIPLIST编码的列表对象
                o->type = OBJ_LIST;
                o->encoding = OBJ_ENCODING_ZIPLIST;
                listTypeConvert(o,OBJ_ENCODING_QUICKLIST);
                break;
            case RDB_TYPE_SET_INTSET: // INTSET编码的集合对象
                o->type = OBJ_SET;
                o->encoding = OBJ_ENCODING_INTSET;
                if (intsetLen(o->ptr) > server.set_max_intset_entries)
                    setTypeConvert(o,OBJ_ENCODING_HT);
                break;
            case RDB_TYPE_ZSET_ZIPLIST: // ZIPLIST编码的有序集合对象
                o->type = OBJ_ZSET;
                o->encoding = OBJ_ENCODING_ZIPLIST;
                if (zsetLength(o) > server.zset_max_ziplist_entries)
                    zsetConvert(o,OBJ_ENCODING_SKIPLIST);
                break;
            case RDB_TYPE_HASH_ZIPLIST: // ZIPLIST编码的哈希对象
                o->type = OBJ_HASH;
                o->encoding = OBJ_ENCODING_ZIPLIST;
                if (hashTypeLength(o) > server.hash_max_ziplist_entries)
                    hashTypeConvert(o, OBJ_ENCODING_HT);
                break;
            default:
                rdbExitReportCorruptRDB("Unknown RDB encoding type %d",rdbtype);
                break;
        }

        // MODULE类型
    } else if (rdbtype == RDB_TYPE_MODULE || rdbtype == RDB_TYPE_MODULE_2) {
        uint64_t moduleid = rdbLoadLen(rdb,NULL);
        moduleType *mt = moduleTypeLookupModuleByID(moduleid);
        char name[10];

        if (rdbCheckMode && rdbtype == RDB_TYPE_MODULE_2)
            return rdbLoadCheckModuleValue(rdb,name);

        if (mt == NULL) {
            moduleTypeNameByID(name,moduleid);
            serverLog(LL_WARNING,"The RDB file contains module data I can't load: no matching module '%s'", name);
            exit(1);
        }
        RedisModuleIO io;
        moduleInitIOContext(io,mt,rdb);
        io.ver = (rdbtype == RDB_TYPE_MODULE) ? 1 : 2;
        /* Call the rdb_load method of the module providing the 10 bit
         * encoding version in the lower 10 bits of the module ID. */
        void *ptr = mt->rdb_load(&io,moduleid&1023);
        if (io.ctx) {
            moduleFreeContext(io.ctx);
            zfree(io.ctx);
        }

        /* Module v2 serialization has an EOF mark at the end. */
        if (io.ver == 2) {
            uint64_t eof = rdbLoadLen(rdb,NULL);
            if (eof != RDB_MODULE_OPCODE_EOF) {
                serverLog(LL_WARNING,"The RDB file contains module data for the module '%s' that is not terminated by the proper module value EOF marker", name);
                exit(1);
            }
        }

        if (ptr == NULL) {
            moduleTypeNameByID(name,moduleid);
            serverLog(LL_WARNING,"The RDB file contains module data for the module type '%s', that the responsible module is not able to load. Check for modules log above for additional clues.", name);
            exit(1);
        }
        o = createModuleObject(mt,ptr);
    } else {
        rdbExitReportCorruptRDB("Unknown RDB encoding type %d",rdbtype);
    }
    return o;
}
```



