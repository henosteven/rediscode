#sentinel 的实现

` 需要准备如下知识点`

* pub | sub 
* timeevent
* TILT模式

基本的数据结构

    typedef struct sentinelAddr {
        char *ip;
        int port;
    } sentinelAddr;

    typedef struct instanceLink {
        int refcount;          /* Number of sentinelRedisInstance owners. */
        int disconnected;      /* Non-zero if we need to reconnect cc or pc. */
        int pending_commands;  /* Number of commands sent waiting for a reply. */
        redisAsyncContext *cc; /* Hiredis context for commands. */
        redisAsyncContext *pc; /* Hiredis context for Pub / Sub. */
        mstime_t cc_conn_time; /* cc connection time. */
        mstime_t pc_conn_time; /* pc connection time. */
        mstime_t pc_last_activity; /* Last time we received any message. */
        mstime_t last_avail_time; /* Last time the instance replied to ping with
                                     a reply we consider valid. */
        mstime_t act_ping_time;   /* Time at which the last pending ping (no pong
                                     received after it) was sent. This field is
                                     set to 0 when a pong is received, and set again
                                     to the current time if the value is 0 and a new
                                     ping is sent. */
        mstime_t last_ping_time;  /* Time at which we sent the last ping. This is
                                     only used to avoid sending too many pings
                                     during failure. Idle time is computed using
                                     the act_ping_time field. */
        mstime_t last_pong_time;  /* Last time the instance replied to ping,
                                     whatever the reply was. That's used to check
                                     if the link is idle and must be reconnected. */
        mstime_t last_reconn_time;  /* Last reconnection attempt performed when
                                           the link was down. */
        } instanceLink;

    typedef struct sentinelRedisInstance {
        int flags;      /* See SRI_... defines */
        char *name;     /* Master name from the point of view of this sentinel. */
        char *runid;    /* Run ID of this instance, or unique ID if is a Sentinel.*/
        uint64_t config_epoch;  /* Configuration epoch. */
        sentinelAddr *addr; /* Master host. ip + port */
        instanceLink *link; /* Link to the instance, may be shared for Sentinels. */
        mstime_t last_pub_time;   /* Last time we sent hello via Pub/Sub. */
        mstime_t last_hello_time; /* Only used if SRI_SENTINEL is set. Last time
                                     we received a hello from this Sentinel
                                     via Pub/Sub. */
        mstime_t last_master_down_reply_time; /* Time of last reply to
                                                 SENTINEL is-master-down command. */
        mstime_t s_down_since_time; /* Subjectively down since time. */
        mstime_t o_down_since_time; /* Objectively down since time. */
        mstime_t down_after_period; /* Consider it down after that period. */
        mstime_t info_refresh;  /* Time at which we received INFO output from it. */

        /* Role and the first time we observed it.
         * This is useful in order to delay replacing what the instance reports
         * with our own configuration. We need to always wait some time in order
         * to give a chance to the leader to report the new configuration before
         * we do silly things. */
        int role_reported;
        mstime_t role_reported_time;
        mstime_t slave_conf_change_time; /* Last time slave master addr changed. */

        /* Master specific. 
         * master对应的slaves和sentinels(其他监控当前master的sentinel)
         */
        dict *sentinels;    /* Other sentinels monitoring the same master. */
        dict *slaves;       /* Slaves for this master instance. */

        unsigned int quorum;/* Number of sentinels that need to agree on failure. */
        int parallel_syncs; /* How many slaves to reconfigure at same time. */
        char *auth_pass;    /* Password to use for AUTH against master & slaves. */

        /* Slave specific. */
        mstime_t master_link_down_time; /* Slave replication link down time. */
        int slave_priority; /* Slave priority according to its INFO output. */
        mstime_t slave_reconf_sent_time; /* Time at which we sent SLAVE OF <new> */
        struct sentinelRedisInstance *master; /* Master instance if it's slave. */
        char *slave_master_host;    /* Master host as reported by INFO */
        int slave_master_port;      /* Master port as reported by INFO */
        int slave_master_link_status; /* Master link status as reported by INFO */
        unsigned long long slave_repl_offset; /* Slave replication offset. */
        /* Failover */
        char *leader;       /* If this is a master instance, this is the runid of
                               the Sentinel that should perform the failover. If
                               this is a Sentinel, this is the runid of the Sentinel
                               that this Sentinel voted as leader. */
        uint64_t leader_epoch; /* Epoch of the 'leader' field. */
        uint64_t failover_epoch; /* Epoch of the currently started failover. */
        int failover_state; /* See SENTINEL_FAILOVER_STATE_* defines. */
        mstime_t failover_state_change_time;
        mstime_t failover_start_time;   /* Last failover attempt start time. */
        mstime_t failover_timeout;      /* Max time to refresh failover state. */
        mstime_t failover_delay_logged; /* For what failover_start_time value we
                                           logged the failover delay. */
        struct sentinelRedisInstance *promoted_slave; /* Promoted slave instance. */
        /* Scripts executed to notify admin or reconfigure clients: when they
         * are set to NULL no script is executed. */
        char *notification_script;
        char *client_reconfig_script;
        sds info; /* cached INFO output */
    } sentinelRedisInstance;

    struct sentinelState {
        char myid[CONFIG_RUN_ID_SIZE+1]; /* This sentinel ID. */
        uint64_t current_epoch;         /* Current epoch. */
        dict *masters;      /* mastername =>sentinelRedisInstance */
        int tilt;           /* Are we in TILT mode, 在sentinelTimer中每次会检查 */
        int running_scripts;    /* Number of scripts in execution right now. */
        mstime_t tilt_start_time;       /* When TITL started. */
        mstime_t previous_time;         /* Last time we ran the time handler. */
        list *scripts_queue;            /* Queue of user scripts to execute. */
        char *announce_ip;  /* IP addr that is gossiped to other sentinels if
                               not NULL. */
        int announce_port;  /* Port that is gossiped to other sentinels if
                               non zero. */
        unsigned long simfailure_flags; /* Failures simulation. */
    } sentinel;


* initSentinelConfig
* initSentinel - 初始化sentinel
* sentinelIsRunning  - 设置MyID并固化到配置文件
* sentinelTimer - 核心逻辑在serverCron中

    run_with_period(100) { 每秒执行的次数为十次
        if (server.sentinel_mode) sentinelTimer();
    }
    
    #define run_with_period(_ms_) if ((_ms_ <= 1000/server.hz) || !(server.cronloops%((_ms_)/(1000/server.hz))))
    解释一下这个run_with_peroid
    server.hz为执行的频率（赫兹）默认为10，也就是每隔100ms执行一次
    server.cronloops在每次执行一次serverCron累加1
    上面的语句翻译为：如果ms小于100毫秒，则直接执行，如果大于100毫秒例如300毫秒，则需要三个周期之后自行
    
    sentinelTimer(void) {
        sentinelCheckTiltCondition(); //对比本次执行的时间与上次执行的时间，如果时差达到触发值，进入TITL模式
        
        /* 非常核心的逻辑都在这个函数里面了 
         * sentinel.masters 在loadconfig的时候已经把masters处理好了
         */
        sentinelHandleDictOfRedisInstances(sentinel.masters);
        
        /* 关于脚本的一些操作 */
        sentinelRunPendingScripts();
        sentinelCollectTerminatedScripts();
        sentinelKillTimedoutScripts();

        /* We continuously change the frequency of the Redis "timer interrupt"
         * in order to desynchronize every Sentinel from every other.
         * This non-determinism avoids that Sentinels started at the same time 
         * exactly continue to stay synchronized asking to be voted at the
         * same time again and again (resulting in nobody likely winning the
         * election because of split brain voting). 
         * henosteven @todo 这里是不是所谓的脑裂问题 
         */   
        server.hz = CONFIG_DEFAULT_HZ + rand() % CONFIG_DEFAULT_HZ;
    }

接下来主要分析sentinelHandleDictOfRedisInstances这个核心处理函数

    void sentinelHandleDictOfRedisInstances(dict *instances) {
        dictIterator *di;
        dictEntry *de;
        sentinelRedisInstance *switch_to_promoted = NULL;

        /* There are a number of things we need to perform against every master. */
        di = dictGetIterator(instances);
        while((de = dictNext(di)) != NULL) {

            /* 取出一个实例进行处理 */
            sentinelRedisInstance *ri = dictGetVal(de);

            /* 处理操作的核心是这个, 包含了所有的核心逻辑*/
            sentinelHandleRedisInstance(ri);

            /* 如果这个实例是master，那么需要取出与他相关联的slaves和sentinels */
            if (ri->flags & SRI_MASTER) {
                sentinelHandleDictOfRedisInstances(ri->slaves);
                sentinelHandleDictOfRedisInstances(ri->sentinels);

                /* 切换主操作标记 */
                if (ri->failover_state == SENTINEL_FAILOVER_STATE_UPDATE_CONFIG) {
                    switch_to_promoted = ri;
                }
            }
        }

        /* 执行切换主操作（其实只是执行更新sentinel的操作，前置的所有操作选主已经弄好了，因为ri已经有了嘛） */
        if (switch_to_promoted)
            sentinelFailoverSwitchToPromotedSlave(switch_to_promoted);
        dictReleaseIterator(di);
    }


> 上面展示了主体的框架逻辑，接下来我们分析sentinelHandleRedisInstance

认识一下下面的四个常量

    #define SENTINEL_INFO_PERIOD 10000   发送info
    #define SENTINEL_PING_PERIOD 1000    sentinelSendPing
    #define SENTINEL_ASK_PERIOD 1000    
    #define SENTINEL_PUBLISH_PERIOD 2000 sentinelSendHello

主观下线：
    sentinelCheckSubjectivelyDown
    核心逻辑就是对比，最后一次pong的时间距当前时间是否已经查过了设定的阈值，如果已经触及，则设置为主观下线

客观下线：
