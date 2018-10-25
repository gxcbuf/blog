---
title: Redis源码阅读(十七) AOF持久化
date: 2018-09-05 15:53:22
tags:
- redis
categories:
- 源码阅读
---

Redis除了RDB持久化外，还提供了AOF持久化，AOF持久化通过保存Redis服务端执行的命令来记录数据库状态。

### 1. AOF文件

```c
// AOF内容格式
命令个数: *<count>\r\n
命令内容: $<length>\r\n<content>\r\n

如 "SET hello world" 指令, count表示命令个数为3，length表示当前命令的长度
"*3\r\n$3\r\nSET\r\n$5\r\nhello\r\n$5\r\nworld\r\n"
这样一条SET指令就如上面一样存储。
```

Redis服务端存储着AOF相关的一些信息:

```c
struct redisServer {
    // ...
     /* AOF persistence */
    // 服务器AOF状态
    int aof_state;                  /* AOF_(ON|OFF|WAIT_REWRITE) */
    // 服务器AOF的fsync策略
    int aof_fsync;                  /* Kind of fsync() policy */
    // 服务器AOF文件名
    char *aof_filename;             /* Name of the AOF file */
    // 如果有aof执行，则不进行fsync
    int aof_no_fsync_on_rewrite;    /* Don't fsync if a rewrite is in prog. */
    // AOF增长比率，默认100
    int aof_rewrite_perc;           /* Rewrite AOF if % growth is > M and... */
    // AOF最小字节数
    off_t aof_rewrite_min_size;     /* the AOF file is at least N bytes. */
    // AOF初始字节数
    off_t aof_rewrite_base_size;    /* AOF size on latest startup or rewrite. */
    // AOF当前字节数
    off_t aof_current_size;         /* AOF current size. */
    // AOF重写提上日程，BGSAVE完成立即执行
    int aof_rewrite_scheduled;      /* Rewrite once BGSAVE terminates. */
    // AOF子进程pid
    pid_t aof_child_pid;            /* PID if rewriting process */
    // AOF缓冲区链表
    list *aof_rewrite_buf_blocks;   /* Hold changes during an AOF rewrite. */
    // AOF缓冲区
    sds aof_buf;      /* AOF buffer, written before entering the event loop */
    // AOF文件描述符
    int aof_fd;       /* File descriptor of currently selected AOF file */
    // AOF选中的db
    int aof_selected_db; /* Currently selected DB in AOF */
    // 延迟执行flush操作的开始时间
    time_t aof_flush_postponed_start; /* UNIX time of postponed AOF flush */
    // 最后一次fsync的直接
    time_t aof_last_fsync;            /* UNIX time of last fsync() */
    // AOF执行最后时间
    time_t aof_rewrite_time_last;   /* Time used by last AOF rewrite run. */
    // AOF执行开始时间
    time_t aof_rewrite_time_start;  /* Current AOF rewrite start time. */
    // AOF执行的状态
    int aof_lastbgrewrite_status;   /* C_OK or C_ERR */
    // 延迟fsync的次数
    unsigned long aof_delayed_fsync;  /* delayed AOF fsync() counter */
    // 重写时是否开启增量式同步，每次写入AOF_AUTOSYNC_BYTES个字节，就执行一次同步
    int aof_rewrite_incremental_fsync;/* fsync incrementally while rewriting? */
    // 上一次AOF操作状态
    int aof_last_write_status;      /* C_OK or C_ERR */
    // 删一次AOF操作错误
    int aof_last_write_errno;       /* Valid if aof_last_write_status is ERR */
    int aof_load_truncated;         /* Don't stop on unexpected AOF EOF. */
    int aof_use_rdb_preamble;       /* Use RDB preamble on AOF rewrites. */

    /* AOF pipes used to communicate between parent and child during rewrite. */
    // 进程通信文件描述描述符，管道
    int aof_pipe_write_data_to_child;
    int aof_pipe_read_data_from_parent;
    int aof_pipe_write_ack_to_parent;
    int aof_pipe_read_ack_from_child;
    int aof_pipe_write_ack_to_child;
    int aof_pipe_read_ack_from_parent;
    int aof_stop_sending_diff;     /* If true stop sending accumulated diffs
                                      to child process. */
    // 保存子进程AOF时累积数据的sds
    sds aof_child_diff;             /* AOF diff accumulator child side. */
    // ...
}
```

### 2. AOF持久化

AOF持久化功能分为追加、写入文件、同步磁盘三个步骤。

#### 2.1 命令追加

当Redis服务器AOF功能开启情况下，服务器执行完一条写命令之后，会将该条命令以AOF格式追加到服务器的AOF缓冲区中。

1. 判断追加命令是否为当前选中数据库，如果不是，则追加SELECT指令
2. 根据不同命令类型分别进行处理，生成AOF格式内容
3. AOF开启状态，将追加到server.aof_buf服务器AOF缓冲区中
4. 如果有AOF子进程正在进行，那么也追加到AOF重写缓冲区中

#### 2.2 写入文件和同步磁盘

Redis服务器进程是一个事件循环，文件事件负责接收客户端的命令请求，以及回复客户端，而时间事件负责执行定时任务，服务器在每次结束事件循环之前，会调用flushAppendOnlyFile函数，判断是否将AOF缓冲区内容写入到AOF文件中。

AOF缓冲区内容写入到文件后是否进行AOF同步有三种策略：

| aof_fsync选项      | 描述                                                         |
| ------------------ | ------------------------------------------------------------ |
| AOF_FSYNC_NO       | 目前为未使用，表示不手动执行fsync，等待操作系统执行          |
| AOF_FSYNC_ALWAYS   | 立即同步到磁盘。                                             |
| AOF_FSYNC_EVERYSEC | 每秒同步一次，该策略下如果后台后fsync执行，那么我们延迟flush操作，最多延迟2秒钟。因为linux上的write(2)操作会被后台的fsync阻塞。 |

> 为了提高文件写入效率，用于调用write函数写入文件时，会将数据写入到内存缓冲区，等待缓冲区被填满或手动执行同步时，才会将内容中同步到磁盘中。

### 3. AOF载入

AOF文件中保存了服务端执行的命令，所以只要执行这些命令，那么服务器的状态就可以被还原。

1. 创建一个不带网络连接的伪客户端，因为Redis命令只能在客户端中执行
2. 循环从AOF文件中读取并解析指令，并在伪客户端中执行

执行完AOF命令，服务器就还原到了之前的状态。

> 服务器在AOF载入阶段，会间歇性处理网络客户端发送的请求，能执行的只有PUBSUB等指令

### 4. AOF重写

AOF持久化是通过保存命令来记录数据库状态，但随着运行时间的延迟，指令的数量会越来越大，而AOF文件体积随着也会变得很大，所以我们通过AOF重写，来减小AOF文件的大小。AOF重写是检查当前数据状态，然后生成新的指令，写入到AOF文件，以这种方式来减少命令数量。

```bash
# 客户端执行以下命令
redis> RPUSH list 1 2
redis> RPUSH list 3 4
redis> RPUSH list 5 6
redis> RPUSH list 7 8	# 1, 2, 3, 4, 5, 6, 7, 8

# AOF重写生成指令
RPUSH list 1 2 3 4 5 6 7 8
```

如果按照普通AOF保存，那么需要保存4条指令，而读取数据库状态后生产新的则只需要一条指令。

AOF重写会进行大量的写操作，所以调用这个函数会阻塞当前线程，而Redis使用单线程处理命令请求，所以AOF重写任务由创建新的子进程执行，这样子进程AOF重写期间，父进程仍然可以处理客户端请求。但这样也会存在一个问题，子进程执行AOF重写期间，父进程会处理请求，这会导致数据库不一致情况发生。

为了解决这种不一致，Redis服务器设置了一个AOF重写缓冲区，当redis服务器执行完命令后，会将这条命令写入AOF缓冲区和AOF重写缓冲区。当子进程AOF重写完成后，向父进程发送一个信号，父进程接收到该信号会调用信号处理函数，将AOF重写缓冲区中的所有内容写入到子进程处理的AOF文件中，这时新AOF文件保存的数据库状态与当前服务器一致。

1. 用户调用BGREWRITEAOF

2. Redis调用rewriteAppendOnlyFileBackground()函数fork子进程

   a. 子进程在临时文件中进行AOF重写

   b. 父进程累积计算差异，追加到AOF重写缓冲区server.aof_rewrite_buf

3. 2a完成后，子进程退出

4. 父进程捕捉子进程信号，将AOF重新缓冲区中记录的差异追加到临时文件，临时文件改名替换真正的AOF文件。

### 5. 源码剖析

- 加载AOF文件

```c
int loadAppendOnlyFile(char *filename) {
    struct client *fakeClient;
    FILE *fp = fopen(filename,"r");
    struct redis_stat sb;
    int old_aof_state = server.aof_state;
    long loops = 0;
    off_t valid_up_to = 0; /* Offset of latest well-formed command loaded. */

    if (fp == NULL) {
        serverLog(LL_WARNING,"Fatal error: can't open the append log file for reading: %s",strerror(errno));
        exit(1);
    }

    /* Handle a zero-length AOF file as a special case. An emtpy AOF file
     * is a valid AOF because an empty server with AOF enabled will create
     * a zero length file at startup, that will remain like that if no write
     * operation is received. */
    // 检查文件正确性
    if (fp && redis_fstat(fileno(fp),&sb) != -1 && sb.st_size == 0) {
        server.aof_current_size = 0;
        fclose(fp);
        return C_ERR;
    }

    /* Temporarily disable AOF, to prevent EXEC from feeding a MULTI
     * to the same file we're about to read. */
    // 临时关闭AOF，防止执行MULTI时，EXEC命令被同步到AOF文件中
    server.aof_state = AOF_OFF;

    fakeClient = createFakeClient();
    // 设置服务器状态
    startLoading(fp);

    /* Check if this AOF file has an RDB preamble. In that case we need to
     * load the RDB file and later continue loading the AOF tail. */
    char sig[5]; /* "REDIS" */
    if (fread(sig,1,5,fp) != 5 || memcmp(sig,"REDIS",5) != 0) {
        /* No RDB preamble, seek back at 0 offset. */
        if (fseek(fp,0,SEEK_SET) == -1) goto readerr;
    } else {
        /* RDB preamble. Pass loading the RDB functions. */
        rio rdb;

        serverLog(LL_NOTICE,"Reading RDB preamble from AOF file...");
        if (fseek(fp,0,SEEK_SET) == -1) goto readerr;
        rioInitWithFile(&rdb,fp);
        if (rdbLoadRio(&rdb,NULL,1) != C_OK) {
            serverLog(LL_WARNING,"Error reading the RDB preamble of the AOF file, AOF loading aborted");
            goto readerr;
        } else {
            serverLog(LL_NOTICE,"Reading the remaining AOF tail...");
        }
    }

    /* Read the actual AOF file, in REPL format, command by command. */
    // 读取AOF文件，并执行命令
    while(1) {
        int argc, j;
        unsigned long len;
        robj **argv;
        char buf[128];
        sds argsds;
        struct redisCommand *cmd;

        /* Serve the clients from time to time */
        // 间隔性处理客户端发送的请求
        // 服务器处于载入状态，能执行的只有PUBSUB
        if (!(loops++ % 1000)) {
            loadingProgress(ftello(fp));
            processEventsWhileBlocked();
        }

        // 读入文件到缓冲区
        if (fgets(buf,sizeof(buf),fp) == NULL) {
            if (feof(fp))
                break;
            else
                goto readerr;
        }
        // 解析文件
        if (buf[0] != '*') goto fmterr;
        if (buf[1] == '\0') goto readerr;
        argc = atoi(buf+1);
        if (argc < 1) goto fmterr;

        argv = zmalloc(sizeof(robj*)*argc);
        fakeClient->argc = argc;
        fakeClient->argv = argv;

        // 挨个解析命令
        for (j = 0; j < argc; j++) {
            if (fgets(buf,sizeof(buf),fp) == NULL) {
                fakeClient->argc = j; /* Free up to j-1. */
                freeFakeClientArgv(fakeClient);
                goto readerr;
            }
            if (buf[0] != '$') goto fmterr;
            len = strtol(buf+1,NULL,10);
            argsds = sdsnewlen(NULL,len);
            if (len && fread(argsds,len,1,fp) == 0) {
                sdsfree(argsds);
                fakeClient->argc = j; /* Free up to j-1. */
                freeFakeClientArgv(fakeClient);
                goto readerr;
            }
            argv[j] = createObject(OBJ_STRING,argsds);
            if (fread(buf,2,1,fp) == 0) {
                fakeClient->argc = j+1; /* Free up to j. */
                freeFakeClientArgv(fakeClient);
                goto readerr; /* discard CRLF */
            }
        }

        /* Command lookup */
        cmd = lookupCommand(argv[0]->ptr);
        if (!cmd) {
            serverLog(LL_WARNING,"Unknown command '%s' reading the append only file", (char*)argv[0]->ptr);
            exit(1);
        }

        /* Run the command in the context of a fake client */
        // 执行命令
        fakeClient->cmd = cmd;
        cmd->proc(fakeClient);

        /* The fake client should not have a reply */
        serverAssert(fakeClient->bufpos == 0 && listLength(fakeClient->reply) == 0);
        /* The fake client should never get blocked */
        serverAssert((fakeClient->flags & CLIENT_BLOCKED) == 0);

        /* Clean up. Command code may have changed argv/argc so we use the
         * argv/argc of the client instead of the local variables. */
        // 释放命令和参数
        freeFakeClientArgv(fakeClient);
        fakeClient->cmd = NULL;
        if (server.aof_load_truncated) valid_up_to = ftello(fp);
    }

    /* This point can only be reached when EOF is reached without errors.
     * If the client is in the middle of a MULTI/EXEC, log error and quit. */
    if (fakeClient->flags & CLIENT_MULTI) goto uxeof;

    // 通过AOF加载完成
loaded_ok: /* DB loaded, cleanup and return C_OK to the caller. */
    fclose(fp);
    freeFakeClient(fakeClient);
    server.aof_state = old_aof_state;
    stopLoading();
    aofUpdateCurrentSize();
    server.aof_rewrite_base_size = server.aof_current_size;
    return C_OK;

    // 各种错误处理

readerr: /* Read error. If feof(fp) is true, fall through to unexpected EOF. */
    if (!feof(fp)) {
        if (fakeClient) freeFakeClient(fakeClient); /* avoid valgrind warning */
        serverLog(LL_WARNING,"Unrecoverable error reading the append only file: %s", strerror(errno));
        exit(1);
    }

uxeof: /* Unexpected AOF end of file. */
    if (server.aof_load_truncated) {
        serverLog(LL_WARNING,"!!! Warning: short read while loading the AOF file !!!");
        serverLog(LL_WARNING,"!!! Truncating the AOF at offset %llu !!!",
            (unsigned long long) valid_up_to);
        if (valid_up_to == -1 || truncate(filename,valid_up_to) == -1) {
            if (valid_up_to == -1) {
                serverLog(LL_WARNING,"Last valid command offset is invalid");
            } else {
                serverLog(LL_WARNING,"Error truncating the AOF file: %s",
                    strerror(errno));
            }
        } else {
            /* Make sure the AOF file descriptor points to the end of the
             * file after the truncate call. */
            if (server.aof_fd != -1 && lseek(server.aof_fd,0,SEEK_END) == -1) {
                serverLog(LL_WARNING,"Can't seek the end of the AOF file: %s",
                    strerror(errno));
            } else {
                serverLog(LL_WARNING,
                    "AOF loaded anyway because aof-load-truncated is enabled");
                goto loaded_ok;
            }
        }
    }
    if (fakeClient) freeFakeClient(fakeClient); /* avoid valgrind warning */
    serverLog(LL_WARNING,"Unexpected end of file reading the append only file. You can: 1) Make a backup of your AOF file, then use ./redis-check-aof --fix <filename>. 2) Alternatively you can set the 'aof-load-truncated' configuration option to yes and restart the server.");
    exit(1);

fmterr: /* Format error. */
    if (fakeClient) freeFakeClient(fakeClient); /* avoid valgrind warning */
    serverLog(LL_WARNING,"Bad file format reading the append only file: make a backup of your AOF file, then use ./redis-check-aof --fix <filename>");
    exit(1);
}
```

- AOF指令追加

```c
void feedAppendOnlyFile(struct redisCommand *cmd, int dictid, robj **argv, int argc) {
    sds buf = sdsempty();
    robj *tmpargv[3];

    /* The DB this command was targeting is not the same as the last command
     * we appended. To issue a SELECT command is needed. */
    // SELECT 命令，确保数据库正确
    if (dictid != server.aof_selected_db) {
        char seldb[64];

        snprintf(seldb,sizeof(seldb),"%d",dictid);
        buf = sdscatprintf(buf,"*2\r\n$6\r\nSELECT\r\n$%lu\r\n%s\r\n",
            (unsigned long)strlen(seldb),seldb);
        server.aof_selected_db = dictid;
    }

    // EXPIRE, PEXPIRE, EXPIREAT命令
    if (cmd->proc == expireCommand || cmd->proc == pexpireCommand ||
        cmd->proc == expireatCommand) {
        /* Translate EXPIRE/PEXPIRE/EXPIREAT into PEXPIREAT */
        buf = catAppendOnlyExpireAtCommand(buf,cmd,argv[1],argv[2]);

        // SETEX 和 PSETEX命令
    } else if (cmd->proc == setexCommand || cmd->proc == psetexCommand) {
        /* Translate SETEX/PSETEX to SET and PEXPIREAT */
        tmpargv[0] = createStringObject("SET",3);
        tmpargv[1] = argv[1];
        tmpargv[2] = argv[3];
        buf = catAppendOnlyGenericCommand(buf,3,tmpargv);
        decrRefCount(tmpargv[0]);
        buf = catAppendOnlyExpireAtCommand(buf,cmd,argv[1],argv[2]);

        // SET 命令
    } else if (cmd->proc == setCommand && argc > 3) {
        int i;
        robj *exarg = NULL, *pxarg = NULL;
        /* Translate SET [EX seconds][PX milliseconds] to SET and PEXPIREAT */
        buf = catAppendOnlyGenericCommand(buf,3,argv);
        for (i = 3; i < argc; i ++) {
            if (!strcasecmp(argv[i]->ptr, "ex")) exarg = argv[i+1];
            if (!strcasecmp(argv[i]->ptr, "px")) pxarg = argv[i+1];
        }
        serverAssert(!(exarg && pxarg));
        if (exarg)
            buf = catAppendOnlyExpireAtCommand(buf,server.expireCommand,argv[1],
                                               exarg);
        if (pxarg)
            buf = catAppendOnlyExpireAtCommand(buf,server.pexpireCommand,argv[1],
                                               pxarg);
        // 其他命令
    } else {
        /* All the other commands don't need translation or need the
         * same translation already operated in the command vector
         * for the replication itself. */
        buf = catAppendOnlyGenericCommand(buf,argc,argv);
    }

    /* Append to the AOF buffer. This will be flushed on disk just before
     * of re-entering the event loop, so before the client will get a
     * positive reply about the operation performed. */
    // 命令追加到AOF缓冲区
    if (server.aof_state == AOF_ON)
        server.aof_buf = sdscatlen(server.aof_buf,buf,sdslen(buf));

    /* If a background append only file rewriting is in progress we want to
     * accumulate the differences between the child DB and the current one
     * in a buffer, so that when the child process will do its work we
     * can append the differences to the new append only file. */
    // 如果有AOF正在进行
    // 则追加到AOF重写缓冲区
    if (server.aof_child_pid != -1)
        aofRewriteBufferAppend((unsigned char*)buf,sdslen(buf));

    sdsfree(buf);
}
```

- 同步AOF到文件

```c
void flushAppendOnlyFile(int force) {
    ssize_t nwritten;
    int sync_in_progress = 0;
    mstime_t latency;

    // 缓冲区没有内容，直接返回
    if (sdslen(server.aof_buf) == 0) return;

    // fsync策略为每秒进行
    if (server.aof_fsync == AOF_FSYNC_EVERYSEC)
        // 是否有后台fsync正在执行
        sync_in_progress = bioPendingJobsOfType(BIO_AOF_FSYNC) != 0;

    // 每秒fsync，但不强制写入
    if (server.aof_fsync == AOF_FSYNC_EVERYSEC && !force) {
        /* With this append fsync policy we do background fsyncing.
         * If the fsync is still in progress we can try to delay
         * the write for a couple of seconds. */
        // 后台有在执行的fsync，我们可以延迟一两秒
        if (sync_in_progress) {
            if (server.aof_flush_postponed_start == 0) {
                /* No previous write postponing, remember that we are
                 * postponing the flush and return. */
                // 之前没有推迟过，则记录推迟时间
                server.aof_flush_postponed_start = server.unixtime;
                return;
            } else if (server.unixtime - server.aof_flush_postponed_start < 2) {
                // 已经推迟过， 但未超过2秒，直接返回
                /* We were already waiting for fsync to finish, but for less
                 * than two seconds this is still ok. Postpone again. */
                return;
            }
            /* Otherwise fall trough, and go write since we can't wait
             * over two seconds. */
            // 有后台fsync执行，并且已经推迟了超过2秒，则执行写操作
            server.aof_delayed_fsync++;
            // 记录日志
            serverLog(LL_NOTICE,"Asynchronous AOF fsync is taking too long (disk is busy?). Writing the AOF buffer without waiting for fsync to complete, this may slow down Redis.");
        }
    }
    /* We want to perform a single write. This should be guaranteed atomic
     * at least if the filesystem we are writing is a real physical one.
     * While this will save us against the server being killed I don't think
     * there is much to do about the whole server stopping for power problems
     * or alike */

    // 执行单个write操作，如果写入物理设备，那么操作应该是原子的
    // 但如果出现类似断电这样的故障，AOF文件也可能出现问题
    // 这是需要redis-check-aof来修复

    // 记录开始时间
    latencyStartMonitor(latency);
    // AOF缓冲区写到AOF文件描述符
    nwritten = aofWrite(server.aof_fd,server.aof_buf,sdslen(server.aof_buf));
    // 记录结束时间
    latencyEndMonitor(latency);

    /* We want to capture different events for delayed writes:
     * when the delay happens with a pending fsync, or with a saving child
     * active, and when the above two conditions are missing.
     * We also use an additional event name to save all samples which is
     * useful for graphing / monitoring purposes. */
    
    if (sync_in_progress) {
        // 后台fsync正在执行
        // 添加采样aof-write-pending-fsync
        latencyAddSampleIfNeeded("aof-write-pending-fsync",latency);
    } else if (server.aof_child_pid != -1 || server.rdb_child_pid != -1) {
        // aof或rdb子进程在执行
        // 添加采样aof-write-active-child
        latencyAddSampleIfNeeded("aof-write-active-child",latency);
    } else {
        // 否则添加aof-write-alone采样
        latencyAddSampleIfNeeded("aof-write-alone",latency);
    }
    // 记录aof-write
    latencyAddSampleIfNeeded("aof-write",latency);

    /* We performed the write so reset the postponed flush sentinel to zero. */
    // 清除延迟write标识
    server.aof_flush_postponed_start = 0;

    // 写入大小不相等
    if (nwritten != (ssize_t)sdslen(server.aof_buf)) {
        static time_t last_write_error_log = 0;
        int can_log = 0;

        /* Limit logging rate to 1 line per AOF_WRITE_LOG_ERROR_RATE seconds. */
        // 日志记录频率限制在每行AOF_WRITE_LOG_ERROR_RATE秒
        if ((server.unixtime - last_write_error_log) > AOF_WRITE_LOG_ERROR_RATE) {
            can_log = 1;
            last_write_error_log = server.unixtime;
        }

        /* Log the AOF write error and record the error code. */
        // 记录AOF写入出错
        if (nwritten == -1) {
            if (can_log) {
                serverLog(LL_WARNING,"Error writing to the AOF file: %s",
                    strerror(errno));
                server.aof_last_write_errno = errno;
            }
        } else {
            if (can_log) {
                serverLog(LL_WARNING,"Short write while writing to "
                                       "the AOF file: (nwritten=%lld, "
                                       "expected=%lld)",
                                       (long long)nwritten,
                                       (long long)sdslen(server.aof_buf));
            }

            if (ftruncate(server.aof_fd, server.aof_current_size) == -1) {
                if (can_log) {
                    serverLog(LL_WARNING, "Could not remove short write "
                             "from the append-only file.  Redis may refuse "
                             "to load the AOF the next time it starts.  "
                             "ftruncate: %s", strerror(errno));
                }
            } else {
                /* If the ftruncate() succeeded we can set nwritten to
                 * -1 since there is no longer partial data into the AOF. */
                nwritten = -1;
            }
            server.aof_last_write_errno = ENOSPC;
        }

        /* Handle the AOF write error. */
        // 处理AOF写入错误
        if (server.aof_fsync == AOF_FSYNC_ALWAYS) {
            /* We can't recover when the fsync policy is ALWAYS since the
             * reply for the client is already in the output buffers, and we
             * have the contract with the user that on acknowledged write data
             * is synced on disk. */
            serverLog(LL_WARNING,"Can't recover from AOF write error when the AOF fsync policy is 'always'. Exiting...");
            exit(1);
        } else {
            /* Recover from failed write leaving data into the buffer. However
             * set an error to stop accepting writes as long as the error
             * condition is not cleared. */
            server.aof_last_write_status = C_ERR;

            /* Trim the sds buffer if there was a partial write, and there
             * was no way to undo it with ftruncate(2). */
            if (nwritten > 0) {
                server.aof_current_size += nwritten;
                sdsrange(server.aof_buf,nwritten,-1);
            }
            return; /* We'll try again on the next call... */
        }
    } else {
        /* Successful write(2). If AOF was in error state, restore the
         * OK state and log the event. */
        // 写入成功，更新状态
        if (server.aof_last_write_status == C_ERR) {
            serverLog(LL_WARNING,
                "AOF write error looks solved, Redis can write again.");
            server.aof_last_write_status = C_OK;
        }
    }
    // 更新AOF文件大小
    server.aof_current_size += nwritten;

    /* Re-use AOF buffer when it is small enough. The maximum comes from the
     * arena size of 4k minus some overhead (but is otherwise arbitrary). */
    // 是否重用缓冲区，AOF缓存小于4K即可重用
    if ((sdslen(server.aof_buf)+sdsavail(server.aof_buf)) < 4000) {
        // 清楚缓冲区内容
        sdsclear(server.aof_buf);
    } else {
        // 释放缓冲区
        sdsfree(server.aof_buf);
        server.aof_buf = sdsempty();
    }

    /* Don't fsync if no-appendfsync-on-rewrite is set to yes and there are
     * children doing I/O in the background. */
    // no-appendfsync-on-rewrite开启或者有AOF，RDB子进程执行，则直接返回，不执行fsync
    if (server.aof_no_fsync_on_rewrite &&
        (server.aof_child_pid != -1 || server.rdb_child_pid != -1))
            return;

    /* Perform the fsync if needed. */
    if (server.aof_fsync == AOF_FSYNC_ALWAYS) {
        /* aof_fsync is defined as fdatasync() for Linux in order to avoid
         * flushing metadata. */
        latencyStartMonitor(latency);
        // 写入到磁盘中
        aof_fsync(server.aof_fd); /* Let's try to get this data on the disk */
        latencyEndMonitor(latency);
        latencyAddSampleIfNeeded("aof-fsync-always",latency);
        // 更新写入时间
        server.aof_last_fsync = server.unixtime;

    } else if ((server.aof_fsync == AOF_FSYNC_EVERYSEC &&
                server.unixtime > server.aof_last_fsync)) {
        // 放到后台执行
        if (!sync_in_progress) aof_background_fsync(server.aof_fd);
        server.aof_last_fsync = server.unixtime;
    }
}

/**
 * 编码命令内容, 按照一定协议记录指令
 * 命令个数：格式: *<count>\r\n
 * 命令内容：格式：$<length>\r\n<content>\r\n
 */
sds catAppendOnlyGenericCommand(sds dst, int argc, robj **argv) {
    char buf[32];
    int len, j;
    robj *o;

    // 命令个数，格式: *<count>\r\n
    buf[0] = '*';
    len = 1+ll2string(buf+1,sizeof(buf)-1,argc);
    buf[len++] = '\r';
    buf[len++] = '\n';
    dst = sdscatlen(dst,buf,len);

    // 命令内容，格式: $<length>\r\n<content>\r\n
    // 如 $3\r\nSET\r\n$5\r\nhello\r\n$5\r\nworld\r\n
    for (j = 0; j < argc; j++) {
        o = getDecodedObject(argv[j]);
        buf[0] = '$';
        len = 1+ll2string(buf+1,sizeof(buf)-1,sdslen(o->ptr));
        buf[len++] = '\r';
        buf[len++] = '\n';
        dst = sdscatlen(dst,buf,len);
        dst = sdscatlen(dst,o->ptr,sdslen(o->ptr));
        dst = sdscatlen(dst,"\r\n",2);
        decrRefCount(o);
    }
    return dst;
}
```

- AOF后台重写

```c
int rewriteAppendOnlyFileBackground(void) {
    pid_t childpid;
    long long start;

    // 已经有AOF或RDB，返回错误
    if (server.aof_child_pid != -1 || server.rdb_child_pid != -1) return C_ERR;

    // 创建进程间通信管道
    if (aofCreatePipes() != C_OK) return C_ERR;

    // 开启子进程到父进程通信的管道
    openChildInfoPipe();

    // 记录开始时间
    start = ustime();

    // 创建子进程
    if ((childpid = fork()) == 0) {
        char tmpfile[256];

        /* Child */
        // 关闭socket文件
        closeListeningSockets(0);
        // 设置进程名
        redisSetProcTitle("redis-aof-rewrite");
        // 临时文件名
        snprintf(tmpfile,256,"temp-rewriteaof-bg-%d.aof", (int) getpid());
        // AOF重写到临时文件
        if (rewriteAppendOnlyFile(tmpfile) == C_OK) {
            // 获取子进程使用内存空间大小
            size_t private_dirty = zmalloc_get_private_dirty(-1);

            // 记录日志
            if (private_dirty) {
                serverLog(LL_NOTICE,
                    "AOF rewrite: %zu MB of memory used by copy-on-write",
                    private_dirty/(1024*1024));
            }

            // 记录子进程使用内存空间
            server.child_info_data.cow_size = private_dirty;
            // 发送数据给父进程
            sendChildInfo(CHILD_INFO_TYPE_AOF);
            // 子进程退出
            exitFromChild(0);
        } else {
            exitFromChild(1);
        }
    } else {
        /* Parent */
        // 记录创建子进程使用时间
        server.stat_fork_time = ustime()-start;
        // 记录创建子进程的速率
        server.stat_fork_rate = (double) zmalloc_used_memory() * 1000000 / server.stat_fork_time / (1024*1024*1024); /* GB per second. */
        // 记录fork使用时间
        latencyAddSampleIfNeeded("fork",server.stat_fork_time/1000);
        // 创建子进程失败，则返回错误
        if (childpid == -1) {
            // 关闭子进程管道
            closeChildInfoPipe();
            // 记录日志
            serverLog(LL_WARNING,
                "Can't rewrite append only file in background: fork: %s",
                strerror(errno));
            // 关闭通信管道
            aofClosePipes();
            return C_ERR;
        }
        // 记录日志
        serverLog(LL_NOTICE,
            "Background append only file rewriting started by pid %d",childpid);
        // 关闭AOF重写计划
        server.aof_rewrite_scheduled = 0;
        // 开始时间
        server.aof_rewrite_time_start = time(NULL);
        // 子进程pid
        server.aof_child_pid = childpid;
        // AOF或RDB期间，不能进行resize
        updateDictResizePolicy();
        /* We set appendseldb to -1 in order to force the next call to the
         * feedAppendOnlyFile() to issue a SELECT command, so the differences
         * accumulated by the parent into server.aof_rewrite_buf will start
         * with a SELECT statement and it will be safe to merge. */
        server.aof_selected_db = -1;
        // 情况脚本缓存
        replicationScriptCacheFlush();
        return C_OK;
    }
    return C_OK; /* unreached */
}
```

