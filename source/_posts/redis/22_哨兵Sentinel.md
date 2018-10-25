---
title: Redis源码阅读(二十二) Sentinel哨兵
date: 2018-10-24 11:13:37
tags:
- redis
categories:
- 源码阅读
---

### 1. 介绍

Sentinel是运行在一个特殊模式下的Redis服务器，由一个或多个服务器构成，它解决了Redis的高可用性，用于监视多个主从服务器，当主服务器下线时会选择该主服务器的从服务器作为新的主节点。

Redis Sentinel提供其他附属任务，如监控，通知，并充当客户端的配置提供程序。

- 监控：Sentinel会不断检查主从节点是否按预期进行工作。
- 通知：Sentinel可以通过API通知系统管理员，另一台受监控的Redis实例出现故障。
- 自动故障转移：如果主节点出现故障，Sentinel可以启动故障转移程序，选择一个从节点升级为主节点，其它从节点重新配置为使用新的主节点，并且使用Redis服务器的应用程序通知有关新服务器的地址和链接。
- 配置提供商：Sentinel充当客户端服务发现的权限来源：客户端链接到Sentinel以询问负责给定服务的当前Redis主节点的地址，如发生故障转移，则报告新地址。

Redis Sentinel是一个分布式系统，可以在多个Sentinel进程中协同工作，当多个Sentinels同意给定主节点不可用时才会执行故障转移，降低误报可能性；即使Sentinel系统中有几个服务器故障，它也能正常工作。

### 2. Sentinel基础

启动Sentinel可以使用`redis-sentinel /path/to/sentinel.conf`或`redis-server /path/to/sentinel.conf --sentinel`命令，这两个命令等效，在运行Sentinel时必须使用配置文件，默认情况下会监听TCP端口26379。

在启动Sentinel必须注意：

1. 至少需要3台Sentinel实例
2. 3台实例必须位于不同的可用区，如物理机或虚拟机，不能在同一台主机
3. Sentinel+Redis分布式系统因为使用异步复制，所以不保证在故障期间保留已确认的写入
4. 需要在客户端支持Sentinel，大多数目前都支持，但仍有些尚未支持
5. **Docker或其他形式的网络地址或端口映射应该谨慎使用**: Docker执行端口重新映射，会打破其他Sentinel进程的Sentinel自动发现以及主服务器的从属列表。

#### 2.1 初始化Sentinel

Sentinel本质是运行在特殊模式下的Redis服务器，所以我们先启动一个Redis服务器，然后让其执行Sentinel部分的代码。

1. Sentinel模式下有特殊的命令表，之前普通服务器的命令表会被清空，换成新的命令表。

    ```c
    /**
     * Sentinel模式下可执行命令
     */
    struct redisCommand sentinelcmds[] = {
        {"ping",pingCommand,1,"",0,NULL,0,0,0,0,0},
        {"sentinel",sentinelCommand,-2,"",0,NULL,0,0,0,0,0},
        {"subscribe",subscribeCommand,-2,"",0,NULL,0,0,0,0,0},
        {"unsubscribe",unsubscribeCommand,-1,"",0,NULL,0,0,0,0,0},
        {"psubscribe",psubscribeCommand,-2,"",0,NULL,0,0,0,0,0},
        {"punsubscribe",punsubscribeCommand,-1,"",0,NULL,0,0,0,0,0},
        {"publish",sentinelPublishCommand,3,"",0,NULL,0,0,0,0,0},
        {"info",sentinelInfoCommand,-1,"",0,NULL,0,0,0,0,0},
        {"role",sentinelRoleCommand,1,"l",0,NULL,0,0,0,0,0},
        {"client",clientCommand,-2,"rs",0,NULL,0,0,0,0,0},
        {"shutdown",shutdownCommand,-1,"",0,NULL,0,0,0,0,0}
    };
    ```

2. 在执行Sentinel代码后，服务器会初始化Sentinel的状态，来专属表示Sentinel功能相关的属性，原有服务器状态仍然会保留。

    ```c
    struct sentinelState {
        // 哨兵ID, 41位
        char myid[CONFIG_RUN_ID_SIZE+1]; /* This sentinel ID. */
        // 当前纪元记录
        uint64_t current_epoch;         /* Current epoch. */
        // 监控的主节点字典，键是主节点实例的名称，值是指向sentinelRedisInstance的指针
        dict *masters;      /* Dictionary of master sentinelRedisInstances.
                               Key is the instance name, value is the
                               sentinelRedisInstance structure pointer. */
        // 是否在TILT模式，该模式仅收集数据，不进行故障转移
        int tilt;           /* Are we in TILT mode? */
        // 当前正在执行的脚本数量
        int running_scripts;    /* Number of scripts in execution right now. */
        // TILT模式开始的时间
        mstime_t tilt_start_time;       /* When TITL started. */
        // 最后一次执行时间出来程序的时间
        mstime_t previous_time;         /* Last time we ran the time handler. */
        // 要执行用户脚本的队列
        list *scripts_queue;            /* Queue of user scripts to execute. */
        // 多个Sentinel之间使用流言来接收关于主服务器是否下线的消息，
        // 并使用投票来决定是否执行自动故障迁移，以及选择哪个从服务器作为新的主服务器
        // 被流言到其他哨兵主机的IP
        char *announce_ip;  /* IP addr that is gossiped to other sentinels if
                               not NULL. */
        // 端口
        int announce_port;  /* Port that is gossiped to other sentinels if
                               non zero. */
        // 故障模拟
        unsigned long simfailure_flags; /* Failures simulation. */
    } sentinel;
    ```

3. Sentinel状态中masters字段存储了当前哨兵监控的master节点的状态信息，它是一个字典，键是被监视的服务器名称，值对应的是下面的结构

   ```c
   typedef struct sentinelRedisInstance {
       // 标记当前Redis实例的类型和状态
       int flags;      /* See SRI_... defines */
       // 从哨兵看来实例的名称
       char *name;     /* Master name from the point of view of this sentinel. */
       // 实例的运行ID，如果是哨兵则为其唯一ID
       char *runid;    /* Run ID of this instance, or unique ID if is a Sentinel.*/
       // 用于实现故障转移，标记新纪元
       uint64_t config_epoch;  /* Configuration epoch. */
       // 实例地址ip和port
       sentinelAddr *addr; /* Master host. */
       // 实例的连接，可能被Sentienls共享
       instanceLink *link; /* Link to the instance, may be shared for Sentinels. */
       // 最后一次通过Pub/Sub发送hello的时间
       mstime_t last_pub_time;   /* Last time we sent hello via Pub/Sub. */
       // 最后一次从Sentienls收到hello的时间 
       mstime_t last_hello_time; /* Only used if SRI_SENTINEL is set. Last time
                                    we received a hello from this Sentinel
                                    via Pub/Sub. */
       // 最后一次回复is-master-down命令的时间
       mstime_t last_master_down_reply_time; /* Time of last reply to
                                                SENTINEL is-master-down command. */
       // 主观下线的时间
       mstime_t s_down_since_time; /* Subjectively down since time. */
       // 客观下线的时间
       mstime_t o_down_since_time; /* Objectively down since time. */
       // 无响应多久后被判断为下线
       mstime_t down_after_period; /* Consider it down after that period. */
       // 收到INFO回复的时间
       mstime_t info_refresh;  /* Time at which we received INFO output from it. */
   
       /* Role and the first time we observed it.
        * This is useful in order to delay replacing what the instance reports
        * with our own configuration. We need to always wait some time in order
        * to give a chance to the leader to report the new configuration before
        * we do silly things. */
       // 实例扮演的角色
       int role_reported;
       // 角色更新的时间
       mstime_t role_reported_time;
       // 最后一次从节点的主节点地址变更的时间
       mstime_t slave_conf_change_time; /* Last time slave master addr changed. */
   
       /* Master specific. 主节点特有属性 */
       // 其他监控相同主节点的Sentienls
       dict *sentinels;    /* Other sentinels monitoring the same master. */
       // 该master节点的从属节点
       dict *slaves;       /* Slaves for this master instance. */
       // 判断该主节点客观下线的投票数
       unsigned int quorum;/* Number of sentinels that need to agree on failure. */
       // 故障转移时，可以同时对新的主节点进行同步的从节点数量
       int parallel_syncs; /* How many slaves to reconfigure at same time. */
       // 连接主节点和从节点的认证密码
       char *auth_pass;    /* Password to use for AUTH against master & slaves. */
   
       /* Slave specific. 从节点特有属性 */
       // 从节点复制操作断开的时间
       mstime_t master_link_down_time; /* Slave replication link down time. */
       // 按照INFO命令输出的从节点优先级
       int slave_priority; /* Slave priority according to its INFO output. */
       // 故障转移时，从节点发送SLAVEOF <new> 命令的时间
       mstime_t slave_reconf_sent_time; /* Time at which we sent SLAVE OF <new> */
       // 从节点保存的主节点实例
       struct sentinelRedisInstance *master; /* Master instance if it's slave. */
       // INFO命令的回复中记录的主节点host
       char *slave_master_host;    /* Master host as reported by INFO */
       // INFO命令的回复中记录的主节点port
       int slave_master_port;      /* Master port as reported by INFO */
       // INFO命令的回复中记录的主从服务器连接状态
       int slave_master_link_status; /* Master link status as reported by INFO */
       // 从节点复制偏移量
       unsigned long long slave_repl_offset; /* Slave replication offset. */
   
       /* Failover 故障转移属性 */
       // 如果是主节点实例，则leader保存的是只想故障转移的Sentinel的runid
       // 如果是哨兵实例，则leader保存的是当前哨兵选举出来的领头runid
       char *leader;       /* If this is a master instance, this is the runid of
                              the Sentinel that should perform the failover. If
                              this is a Sentinel, this is the runid of the Sentinel
                              that this Sentinel voted as leader. */
       // leader的纪元
       uint64_t leader_epoch; /* Epoch of the 'leader' field. */
       // 当前执行故障转移的纪元
       uint64_t failover_epoch; /* Epoch of the currently started failover. */
       // 故障转移操作的状态
       int failover_state; /* See SENTINEL_FAILOVER_STATE_* defines. */
       // 状态改变世界
       mstime_t failover_state_change_time;
       // 最后一次尝试故障转移开始时间
       mstime_t failover_start_time;   /* Last failover attempt start time. */
       // 更新故障转移状态的最大超时时间
       mstime_t failover_timeout;      /* Max time to refresh failover state. */
       // 记录故障转移延迟的时间
       mstime_t failover_delay_logged; /* For what failover_start_time value we
                                          logged the failover delay. */
       // 晋身为新主节点的从节点实例
       struct sentinelRedisInstance *promoted_slave; /* Promoted slave instance. */
       /* Scripts executed to notify admin or reconfigure clients: when they
        * are set to NULL no script is executed. */
       // 通知admin的可执行脚本的地址，如果为空，则表示没有执行的脚本
       char *notification_script;
       // 通知配置的client的可执行脚本的地址，如果为空，则表示没有执行的脚本
       char *client_reconfig_script;
       // 缓存INFO命令的输出
       sds info; /* cached INFO output */
   } sentinelRedisInstance;
   ```

4. 初始化Sentinel最后一步会创建面向被监视主服务器的网络连接，Sentinel将成为主服务器的客户端，可以向主服务器发送命令并获取相关信息。对于每个被Sentinel监视的服务器，Sentinel会创建两个异步网络连接：
   - 命令连接：用于向主服务器发送命令，并接受回复消息
   - 订阅连接：用于订阅主服务器`__sentinel__:hello`频道

#### 2.2 与服务器间的通信

1. 获取主从服务器实例状态

   Sentinel默认每10秒发送INFO命令给被监控的主服务器，通过INFO命令的回复获得主节点的状态数据，通过分析可获取下面信息：

   - 关于主服务器本身的信息，包括服务器运行ID，和服务器角色。
   - 主服务器属下所有从服务器的信息，每个记录如`slave:ip=1.2.3.4,port=123,state=online`

   根据这些信息，Sentinel将会对主节点的信息进行更新。

	> 获取从节点ip和端口后，Sentinel也会每10秒发送INFO命令来获取从节点信息。

	当Sentinel发现主节点有新从节点加入时，Sentinel会创建该新实例的相应结构并创建命令连接和订阅连接。创建命令连接后，默认每10秒与该实例通过INFO获取一次状态信息。

2. 向主从服务器发送信息

   默认情况下，Sentinel每两秒通过命令连接向所有被监视的主从节点发送`PUBLISH __sentinel__:hello "<sentinel_ip>,<sentinel_port>,<sentinel_runid>,<current_epoch>,<master_name>,<master_ip>,<master_ip>,<master_port>,<master_config_epoch>"`

   ```c
   int sentinelSendHello(sentinelRedisInstance *ri) {
       char ip[NET_IP_STR_LEN];
       char payload[NET_IP_STR_LEN+1024];
       int retval;
       char *announce_ip;
       int announce_port;
       // 主服务器实例
       sentinelRedisInstance *master = (ri->flags & SRI_MASTER) ? ri : ri->master;
       // 主服务器地址
       sentinelAddr *master_addr = sentinelGetCurrentMasterAddress(master);
   
       // 连接关闭则返回C_ERR
       if (ri->link->disconnected) return C_ERR;
   
       /* Use the specified announce address if specified, otherwise try to
        * obtain our own IP address. */
       // 如果Sentinel指定了announce则使用它
       if (sentinel.announce_ip) {
           announce_ip = sentinel.announce_ip;
       } else {
           if (anetSockName(ri->link->cc->c.fd,ip,sizeof(ip),NULL) == -1)
               return C_ERR;
           announce_ip = ip;
       }
       announce_port = sentinel.announce_port ?
                       sentinel.announce_port : server.port;
   
       /* Format and send the Hello message. */
       // 格式化Hello消息
       snprintf(payload,sizeof(payload),
           "%s,%d,%s,%llu," /* Info about this sentinel. */
           "%s,%s,%d,%llu", /* Info about current master. */
           announce_ip, announce_port, sentinel.myid,
           (unsigned long long) sentinel.current_epoch,
           /* --- */
           master->name,master_addr->ip,master_addr->port,
           (unsigned long long) master->config_epoch);
       // 异步执行命令
       retval = redisAsyncCommand(ri->link->cc,
           sentinelPublishReplyCallback, ri, "PUBLISH %s %s",
               SENTINEL_HELLO_CHANNEL,payload);
       if (retval != C_OK) return C_ERR;
       ri->link->pending_commands++;
       return C_OK;
   }
   ```

3. 接收主从服务器消息

   当Sentinel与主从服务器建立订阅连接后，就会订阅`__sentinel__:hello`频道，通过该频道获取来自服务器的消息，也就表明每个Sentinel即通过命令连接向服务器的`__sentinel__:hello`发送消息，又订阅服务器的`__sentinel__:hello`接收其消息。

   对监视同一个服务器的多个Sentinel来说，他们会接收到来自不同Sentinel的信息。

   ```c
   void sentinelProcessHelloMessage(char *hello, int hello_len) {
       /* Format is composed of 8 tokens:
        * 0=ip,1=port,2=runid,3=current_epoch,4=master_name,
        * 5=master_ip,6=master_port,7=master_config_epoch. */
       int numtokens, port, removed, master_port;
       uint64_t current_epoch, master_config_epoch;
       char **token = sdssplitlen(hello, hello_len, ",", 1, &numtokens);
       sentinelRedisInstance *si, *master;
   
       if (numtokens == 8) {
           /* Obtain a reference to the master this hello message is about */
           // 获取主服务器名称，丢到未知的master数据
           master = sentinelGetMasterByName(token[4]);
           if (!master) goto cleanup; /* Unknown master, skip the message. */
   
           /* First, try to see if we already have this sentinel. */
           // 获取消息内容，并转化为相应类型
           port = atoi(token[1]);
           master_port = atoi(token[6]);
           si = getSentinelRedisInstanceByAddrAndRunID(
                           master->sentinels,token[0],port,token[2]);
           current_epoch = strtoull(token[3],NULL,10);
           master_config_epoch = strtoull(token[7],NULL,10);
   
           // 如果没在主节点的Sentinel字典中找到该Sentinel
           if (!si) {
               /* If not, remove all the sentinels that have the same runid
                * because there was an address change, and add the same Sentinel
                * with the new address back. */
               // 删除master中所有具有相同runid的Sentinel节点, 并添加具有
               // 新地址的同一Sentinel实例
               removed = removeMatchingSentinelFromMaster(master,token[2]);
               if (removed) {
                   sentinelEvent(LL_NOTICE,"+sentinel-address-switch",master,
                       "%@ ip %s port %d for %s", token[0],port,token[2]);
               } else {
                   /* Check if there is another Sentinel with the same address this
                    * new one is reporting. What we do if this happens is to set its
                    * port to 0, to signal the address is invalid. We'll update it
                    * later if we get an HELLO message. */
                   // 检查是否存在和Hello信息中报告地址一致的Sentinel，如果发生该情况
                   // 我们将端口设置为0，以表示地址无效，如果过会收到HELLO消息, 我们则更新它
                   sentinelRedisInstance *other =
                       getSentinelRedisInstanceByAddrAndRunID(
                           master->sentinels, token[0],port,NULL);
                   if (other) {
                       sentinelEvent(LL_NOTICE,"+sentinel-invalid-addr",other,"%@");
                       other->addr->port = 0; /* It means: invalid address. */
                       sentinelUpdateSentinelAddressInAllMasters(other);
                   }
               }
   
               /* Add the new sentinel. */
               // 添加一个新的Sentinel结构, 添加到master的Sentinel中
               si = createSentinelRedisInstance(token[2],SRI_SENTINEL,
                               token[0],port,master->quorum,master);
   
               // 添加成功
               if (si) {
                   // 若未删除之前的Sentinel，发生事件通知
                   if (!removed) sentinelEvent(LL_NOTICE,"+sentinel",si,"%@");
                   /* The runid is NULL after a new instance creation and
                    * for Sentinels we don't have a later chance to fill it,
                    * so do it now. */
                   // 更新runid
                   si->runid = sdsnew(token[2]);
                   // 尝试与其他Sentinel共享连接
                   sentinelTryConnectionSharing(si);
                   // 若已删除了Sentinel节点，更新其他的Sentinel信息
                   if (removed) sentinelUpdateSentinelAddressInAllMasters(si);
                   // 刷新配置
                   sentinelFlushConfig();
               }
           }
   
           /* Update local current_epoch if received current_epoch is greater.*/
           // 更新更改的纪元信息
           if (current_epoch > sentinel.current_epoch) {
               sentinel.current_epoch = current_epoch;
               sentinelFlushConfig();
               sentinelEvent(LL_WARNING,"+new-epoch",master,"%llu",
                   (unsigned long long) sentinel.current_epoch);
           }
   
           /* Update master info if received configuration is newer. */
           // 更新接收到的新配置信息
           if (si && master->config_epoch < master_config_epoch) {
               master->config_epoch = master_config_epoch;
               if (master_port != master->addr->port ||
                   strcmp(master->addr->ip, token[5]))
               { sentinelAddr *old_addr;
   
                   sentinelEvent(LL_WARNING,"+config-update-from",si,"%@");
                   sentinelEvent(LL_WARNING,"+switch-master",
                       master,"%s %s %d %s %d",
                       master->name,
                       master->addr->ip, master->addr->port,
                       token[5], master_port);
   
                   old_addr = dupSentinelAddr(master->addr);
                   sentinelResetMasterAndChangeAddress(master, token[5], master_port);
                   sentinelCallClientReconfScript(master,
                       SENTINEL_OBSERVER,"start",
                       old_addr,master->addr);
                   releaseSentinelAddr(old_addr);
               }
           }
   
           /* Update the state of the Sentinel. */
           // 更新最后一次接收到Sentinel的hello消息时间
           if (si) si->last_hello_time = mstime();
       }
   
   cleanup:
       sdsfreesplitres(token,numtokens);
   }
   ```


### 3. 监测服务器下线状态

#### 3.1 主观下线状态

默认情况下，Sentinel会每秒向所有与它建立命令连接的实例发送PING命令，并根据返回信息判断实例是否处于在线状态。接收到PING命令的回复若为`+PONG -LOADING -MASTERDOWN`则会认为是正常回复，会更新最后接收到PING命令的时间，但如果是`BUSY`则表示非正常回复。

> 当Lua脚本运行的时间超过配置的Lua脚本时间限制时，Redis实例会返回-BUSY错误。在触发故障转移之前发生这种情况时，Redis Sentinel将尝试发送[SCRIPT KILL](https://redis.io/commands/script-kill) 命令，该命令仅在脚本为只读时才会成功。
>
> 如果在尝试之后实例仍然处于错误状态，则最终将进行故障转移。

在低活跃状态下，我们需要断开命令连接和订阅连接。

如果主节点长时间没有回复或无效回复，或者Sentinel认为服务器时主节点，但它自己上报为从节点，那么会将该实例设置为主观下线状态。

> 判断时间可以通过Sentinel的配置文件中的down-after-milliseconds参数进行修改

```c
void sentinelCheckSubjectivelyDown(sentinelRedisInstance *ri) {
    mstime_t elapsed = 0;

    // 获取距离服务器最后一次响应已经过去多长时间
    // 正常获取距离最后一次PING的时间，丢失连接则获取最后一次可用时间
    if (ri->link->act_ping_time)
        elapsed = mstime() - ri->link->act_ping_time;
    else if (ri->link->disconnected)
        elapsed = mstime() - ri->link->last_avail_time;

    /* Check if we are in need for a reconnection of one of the
     * links, because we are detecting low activity.
     *
     * 检查我们是否需要重新连接其中的一个链接，因为我们处于低活动状态
     * 1) 检查命令连接是否已连接，若连接已超过1.5s，且发送过PING命令
     *    但连接活跃度很低，那么久断开实例的cc命令连接
     *
     * 1) Check if the command link seems connected, was connected not less
     *    than SENTINEL_MIN_LINK_RECONNECT_PERIOD, but still we have a
     *    pending ping for more than half the timeout. */
    if (ri->link->cc &&
        (mstime() - ri->link->cc_conn_time) >
        SENTINEL_MIN_LINK_RECONNECT_PERIOD &&
        ri->link->act_ping_time != 0 && /* Ther is a pending ping... */
        /* The pending ping is delayed, and we did not received
         * error replies as well. */
        (mstime() - ri->link->act_ping_time) > (ri->down_after_period/2) &&
        (mstime() - ri->link->last_pong_time) > (ri->down_after_period/2))
    {
        instanceLinkCloseConnection(ri->link,ri->link->cc);
    }

    /* 2) Check if the pubsub link seems connected, was connected not less
     *    than SENTINEL_MIN_LINK_RECONNECT_PERIOD, but still we have no
     *    activity in the Pub/Sub channel for more than
     *    SENTINEL_PUBLISH_PERIOD * 3.
     *
     * 2) 检测pc订阅连接是否也处于低活跃状态，则断开订阅连接
     */
    if (ri->link->pc &&
        (mstime() - ri->link->pc_conn_time) >
         SENTINEL_MIN_LINK_RECONNECT_PERIOD &&
        (mstime() - ri->link->pc_last_activity) > (SENTINEL_PUBLISH_PERIOD*3))
    {
        instanceLinkCloseConnection(ri->link,ri->link->pc);
    }

    /* Update the SDOWN flag. We believe the instance is SDOWN if:
     * 更新主观下线标志，条件：
     * 1) 没有回复命令
     * 2) Sentinel认为服务器是主节点，但它自己报告的是从节点
     *
     * 1) It is not replying.
     * 2) We believe it is a master, it reports to be a slave for enough time
     *    to meet the down_after_period, plus enough time to get two times
     *    INFO report from the instance. */
    if (elapsed > ri->down_after_period ||
        (ri->flags & SRI_MASTER &&
         ri->role_reported == SRI_SLAVE &&
         mstime() - ri->role_reported_time >
          (ri->down_after_period+SENTINEL_INFO_PERIOD*2)))
    {
        /* Is subjectively down */
        // 设置主观下线标识
        if ((ri->flags & SRI_S_DOWN) == 0) {
            // 发送"+sdown"事件通知
            sentinelEvent(LL_WARNING,"+sdown",ri,"%@");
            // 记录主观下线时间和标识
            ri->s_down_since_time = mstime();
            ri->flags |= SRI_S_DOWN;
        }
    } else {
        /* Is subjectively up */
        // 如果设置了主观下线标识，则取消标识
        if (ri->flags & SRI_S_DOWN) {
            sentinelEvent(LL_WARNING,"-sdown",ri,"%@");
            ri->flags &= ~(SRI_S_DOWN|SRI_SCRIPT_KILL_SENT);
        }
    }
}
```

#### 3.2 客观下线状态

当Sentinel将一个主服务器判断为主观下线状态后，为确认是否真的下线，会与监视该主服务器的Sentinel们进行询问，以确定主服务器状态。

```c
/**
 * 客观下线状态检测, 根据配置的投票数判断
 * 注意：ODOWN是一个较弱的投票，它仅表示在给定时间范围内报告的实例无法
 *       访问足够的Sentinel，然而，消息可以被延迟，所以并不能保证N各实例
 *       同时同意关闭状态。
 */
void sentinelCheckObjectivelyDown(sentinelRedisInstance *master) {
    dictIterator *di;
    dictEntry *de;
    unsigned int quorum = 0, odown = 0;

    // 该master已经被判定为客观下线
    if (master->flags & SRI_S_DOWN) {
        /* Is down for enough sentinels? */
        // 投票
        quorum = 1; /* the current sentinel. */
        /* Count all the other sentinels. */
        // 遍历监控该节点的所有Sentinel
        di = dictGetIterator(master->sentinels);
        while((de = dictNext(di)) != NULL) {
            sentinelRedisInstance *ri = dictGetVal(de);

            // 如果Sentinel认为其下线，则投票数+1
            if (ri->flags & SRI_MASTER_DOWN) quorum++;
        }
        dictReleaseIterator(di);
        // 如果投票数超过设置的客观下线投票数，则客观下线
        if (quorum >= master->quorum) odown = 1;
    }

    /* Set the flag accordingly to the outcome. */
    // 设置客观下线标识
    if (odown) {
        if ((master->flags & SRI_O_DOWN) == 0) {
            sentinelEvent(LL_WARNING,"+odown",master,"%@ #quorum %d/%d",
                quorum, master->quorum);
            master->flags |= SRI_O_DOWN;
            master->o_down_since_time = mstime();
        }

        // 如果不用设置客观下线标识
    } else {
        // 如果已经是客观下线，的取消客观下线表示
        if (master->flags & SRI_O_DOWN) {
            // 发送"-odown"事件通知
            sentinelEvent(LL_WARNING,"-odown",master,"%@");
            master->flags &= ~SRI_O_DOWN;
        }
    }
}

/* Receive the SENTINEL is-master-down-by-addr reply, see the
 * sentinelAskMasterStateToOtherSentinels() function for more information. */
/**
 * 接收Sentinel关于is-master-down-by-addr的回复
 */
void sentinelReceiveIsMasterDownReply(redisAsyncContext *c, void *reply, void *privdata) {
    sentinelRedisInstance *ri = privdata;
    instanceLink *link = c->data;
    redisReply *r;

    if (!reply || !link) return;
    // 已发送未回复命令数-1
    link->pending_commands--;
    r = reply;

    /* Ignore every error or unexpected reply.
     * Note that if the command returns an error for any reason we'll
     * end clearing the SRI_MASTER_DOWN flag for timeout anyway. */
    // 忽略错误回复, 注意：如果回复错误，那么需要清除SRI_MASTER_DOWN标志
    if (r->type == REDIS_REPLY_ARRAY && r->elements == 3 &&
        r->element[0]->type == REDIS_REPLY_INTEGER &&
        r->element[1]->type == REDIS_REPLY_STRING &&
        r->element[2]->type == REDIS_REPLY_INTEGER)
    {
        // 最后一次回复下线时间
        ri->last_master_down_reply_time = mstime();
        // 如果回复的第一个元素是1，表示同意客观下线
        if (r->element[0]->integer == 1) {
            // 设置客观下线
            ri->flags |= SRI_MASTER_DOWN;
        } else {
            // 取消客观下线
            ri->flags &= ~SRI_MASTER_DOWN;
        }
        // 如果回复的第一个元素是"*"
        if (strcmp(r->element[1]->str,"*")) {
            /* If the runid in the reply is not "*" the Sentinel actually
             * replied with a vote. */

            // 释放leader
            sdsfree(ri->leader);
            // 打印日志
            if ((long long)ri->leader_epoch != r->element[2]->integer)
                serverLog(LL_WARNING,
                    "%s voted for %s %llu", ri->name,
                    r->element[1]->str,
                    (unsigned long long) r->element[2]->integer);
            // 设置leader和其纪元
            ri->leader = sdsnew(r->element[1]->str);
            ri->leader_epoch = r->element[2]->integer;
        }
    }
}

/* If we think the master is down, we start sending
 * SENTINEL IS-MASTER-DOWN-BY-ADDR requests to other sentinels
 * in order to get the replies that allow to reach the quorum
 * needed to mark the master in ODOWN state and trigger a failover. */
#define SENTINEL_ASK_FORCED (1<<0)
/**
 * 如果Sentinel认为主节点下线，会发送一个SENTINEL IS-MASTER-DOWN-BY-ADDR给所有的Sentinel以
 * 获取回复，尝试获取足够多的票数，标记主节点为客观下线状态用于触发故障转移
 */
void sentinelAskMasterStateToOtherSentinels(sentinelRedisInstance *master, int flags) {
    dictIterator *di;
    dictEntry *de;

    di = dictGetIterator(master->sentinels);
    // 遍历监控master的所有Sentinel节点
    while((de = dictNext(di)) != NULL) {
        sentinelRedisInstance *ri = dictGetVal(de);
        // 获取距离当前Sentinel实例最后一次回复该命令所过去的时间
        mstime_t elapsed = mstime() - ri->last_master_down_reply_time;
        char port[32];
        int retval;

        /* If the master state from other sentinel is too old, we clear it. */
        // 如果master的状态太久没有更新，那么则清除
        if (elapsed > SENTINEL_ASK_PERIOD*5) {
            ri->flags &= ~SRI_MASTER_DOWN;
            sdsfree(ri->leader);
            ri->leader = NULL;
        }

        /* Only ask if master is down to other sentinels if:
         * 满足以下条件则向其他Sentinel询问主节点是否下线
         * 1) 当前Sentinel节点认为它已经下线，并且处于故障转移状态
         * 2) 其他Sentinel与当前Sentinel保持连接状态
         * 3) 在SENTINEL_ASK_PERIOD毫秒内没有收到INFO回复
         *
         * 1) We believe it is down, or there is a failover in progress.
         * 2) Sentinel is connected.
         * 3) We did not received the info within SENTINEL_ASK_PERIOD ms. */
        // 主节点没有处于客观下线状态，则跳过当前Sentinel节点
        if ((master->flags & SRI_S_DOWN) == 0) continue;
        if (ri->link->disconnected) continue;
        if (!(flags & SENTINEL_ASK_FORCED) &&
            mstime() - ri->last_master_down_reply_time < SENTINEL_ASK_PERIOD)
            continue;

        /* Ask */
        ll2string(port,sizeof(port),master->addr->port);
        // 异步发送命令
        retval = redisAsyncCommand(ri->link->cc,
                    sentinelReceiveIsMasterDownReply, ri,
                    "SENTINEL is-master-down-by-addr %s %s %llu %s",
                    master->addr->ip, port,
                    sentinel.current_epoch,
                    (master->failover_state > SENTINEL_FAILOVER_STATE_NONE) ?
                    sentinel.myid : "*");
        if (retval == C_OK) ri->link->pending_commands++;
    }
    dictReleaseIterator(di);
}
```

### 4. Sentinel选举

当一个主服务器被判定为主观下线，监视这个服务器的所有Sentinel会协商一个leader，并由leader对主服务器进行故障转移操作，以下是leader选取的规则：

- 所有在线Sentinel都被设置为候选人，即每个Sentinel都有资格成为leader
- 每次选举之后，所有Sentinel的纪元加1
- 在一个纪元里，每个Sentinel都有一次将某个Sentinel设置为leader的机会，并且该纪元内不能改变
- 每个发现主节点客观下线的Sentinel都会要求其他Sentinel将自己设为leader
- 当一个Sentinel向另一个Sentinel发送`SENTINEL is-master-down-by-addr`命令，且命令中runid参数不是*而是Sentinel的runid那么表示要求目标Sentinel将leader设置为它
- Sentinel设置leader的规则就是先到先得，后来的投票请求会被拒绝
- 如果某个Sentinel获得了半数以上(50% + 1)的选票，那么它就会成为该纪元的leader
- 如果给定时间内未选出leader，那么会隔一段时间重新选举，直到leader出现

> 该算法类似Raft的一致性算法

### 5. 故障转移

在选择出领头Sentinel之后，由其负责对已下线的主节点进行故障转移操作：

1. 在已下线的主节点的从节点中选择一个并将其转换为主节点

   选出服务器后向其发送`SLAVEOF no one`命令，将其转换为主服务器，在发送该命令后，领头Sentinel每秒向服务器发送INFO命令并观察回复信息，当其中role角色从slave转换为master时就表示成功。

   > 新主节点挑选原则：
   >
   > 1) 删除所有下线的服务器，保证列表中都是正常在线的
   >
   > 2) 删除列表中5秒内未回复INFO命令的服务器
   >
   > 3) 删除所有与已下线主节点断开连接超过`down-after-milliseconds * 10`的服务器
   >
   > 之后根据服务器优先级选择，若优先级相同，则选择偏移量最大的，若偏移量也相同，则选择runid最小的服务器

2. 让已下线主节点的所有从节点改为复制新的主节点

3. 将已下线的主节点设为新主节点的从节点，当该节点重新上线时，会成为新主节点的从节点

### 6. TILT模式

TILT模式是一种特殊的"保护模式"，当检测到奇怪的状态，Sentinel可以进入该模式。

Sentinel定时器中断每秒调用10次，所以预期两次调用直接会经过100毫秒，但由于系统某些原因可能会导致大于100毫秒，如果时差为负或者大于2秒，那么则进入TILT模式。

在TILT模式下，Sentinel会继续监控服务器，但是：

- 不会执行任何操作，例如故障转移
- 对`SENTINEL is-master-down-by-addr`回复负数，因为不信任故障检测功能

如果TILT正常运行30秒，那么则退出该模式。