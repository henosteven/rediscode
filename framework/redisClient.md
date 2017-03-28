## redisClient的实现

#### 未完待续

` 每次有客户端连接服务器的时候，服务器会给每个客户端生成一个redisClient结构
  还有的fakeClient，在恢复数据和script中会使用到，相当于是没有连接的客户端
`

> 首先我们来看看redisClient数据结构

    typedef struct client {
        uint64_t id;            /* Client incremental unique ID. */
        int fd;                 /* Client socket. */
        redisDb *db;            /* Pointer to currently SELECTed DB. */
        int dictid;             /* ID of the currently SELECTed DB. */
        robj *name;             /* As set by CLIENT SETNAME. */
        sds querybuf;           /* Buffer we use to accumulate client queries. */
        size_t querybuf_peak;   /* Recent (100ms or more) peak of querybuf size. */
        int argc;               /* Num of arguments of current command. */
        robj **argv;            /* Arguments of current command. */
        struct redisCommand *cmd, *lastcmd;  /* Last command executed. */
        int reqtype;            /* Request protocol type: PROTO_REQ_* */
        int multibulklen;       /* Number of multi bulk arguments left to read. */
        long bulklen;           /* Length of bulk argument in multi bulk request. */
        list *reply;            /* List of reply objects to send to the client. */
        unsigned long long reply_bytes; /* Tot bytes of objects in reply list. */
        size_t sentlen;         /* Amount of bytes already sent in the current
                                   buffer or object being sent. */
        time_t ctime;           /* Client creation time. */
        time_t lastinteraction; /* Time of the last interaction, used for timeout */
        time_t obuf_soft_limit_reached_time;

        /* 记录了客户端当前的状态（状态集合） */
        int flags;              /* Client flags: CLIENT_* macros. */
        int authenticated;      /* When requirepass is non-NULL. */
        int replstate;          /* Replication state if this is a slave. */
        int repl_put_online_on_ack; /* Install slave write handler on ACK. */
        int repldbfd;           /* Replication DB file descriptor. */
        off_t repldboff;        /* Replication DB file offset. */
        off_t repldbsize;       /* Replication DB file size. */
        sds replpreamble;       /* Replication DB preamble. */
        long long reploff;      /* Replication offset if this is our master. */
        long long repl_ack_off; /* Replication ack offset, if this is a slave. */
        long long repl_ack_time;/* Replication ack time, if this is a slave. */
        long long psync_initial_offset; /* FULLRESYNC reply offset other slaves
                                           copying this slave output buffer
                                           should use. */
        char replrunid[CONFIG_RUN_ID_SIZE+1]; /* Master run id if is a master. */
        int slave_listening_port; /* As configured with: REPLCONF listening-port */
        char slave_ip[NET_IP_STR_LEN]; /* Optionally given by REPLCONF ip-address */
        int slave_capa;         /* Slave capabilities: SLAVE_CAPA_* bitwise OR. */

        //记录了当前client事务状态
        multiState mstate;      /* MULTI/EXEC state */
        int btype;              /* Type of blocking op if CLIENT_BLOCKED. */
        blockingState bpop;     /* blocking state 存放clinet的block的key信息*/
        long long woff;         /* Last write global replication offset. */

        //记录当前client watch的keys
        list *watched_keys;     /* Keys WATCHED for MULTI/EXEC CAS */

        //记录当前client订阅的频道
        dict *pubsub_channels;  /* channels a client is interested in (SUBSCRIBE) */

        //记录当前client订阅的频道（模式）
        list *pubsub_patterns;  /* patterns a client is interested in (SUBSCRIBE) */
        sds peerid;             /* Cached peer ID. */

        /* Response buffer */
        int bufpos;
        char buf[PROTO_REPLY_CHUNK_BYTES];
    } client;

客户端有很多标记

    /* Client flags */
    #define CLIENT_SLAVE (1<<0)   /* This client is a slave server */
    #define CLIENT_MASTER (1<<1)  /* This client is a master server */
    #define CLIENT_MONITOR (1<<2) /* This client is a slave monitor, see MONITOR */
    #define CLIENT_MULTI (1<<3)   /* This client is in a MULTI context */
    #define CLIENT_BLOCKED (1<<4) /* The client is waiting in a blocking operation */

    /*
     * 当当前client watch的key发生变化的时候
     */
    #define CLIENT_DIRTY_CAS (1<<5) /* Watched keys modified. EXEC will fail. */
    #define CLIENT_CLOSE_AFTER_REPLY (1<<6) /* Close after writing entire reply. */
    #define CLIENT_UNBLOCKED (1<<7) /* This client was unblocked and is stored in
                                      server.unblocked_clients */
    #define CLIENT_LUA (1<<8) /* This is a non connected client used by Lua */
    #define CLIENT_ASKING (1<<9)     /* Client issued the ASKING command */
    #define CLIENT_CLOSE_ASAP (1<<10)/* Close this client ASAP */
    #define CLIENT_UNIX_SOCKET (1<<11) /* Client connected via Unix domain socket */

    /* 当命令参数不对，命令不存在，命令会对内存影响而当前内存不足等情况下
     * 在事物执行exec的会有影响
     */
    #define CLIENT_DIRTY_EXEC (1<<12)  /* EXEC will fail for errors while queueing */
    #define CLIENT_MASTER_FORCE_REPLY (1<<13)  /* Queue replies even if is master */
    #define CLIENT_FORCE_AOF (1<<14)   /* Force AOF propagation of current cmd. */
    #define CLIENT_FORCE_REPL (1<<15)  /* Force replication of current cmd. */
    #define CLIENT_PRE_PSYNC (1<<16)   /* Instance don't understand PSYNC. */
    #define CLIENT_READONLY (1<<17)    /* Cluster client is in read-only state. */
    #define CLIENT_PUBSUB (1<<18)      /* Client is in Pub/Sub mode. */
    #define CLIENT_PREVENT_AOF_PROP (1<<19)  /* Don't propagate to AOF. */
    #define CLIENT_PREVENT_REPL_PROP (1<<20)  /* Don't propagate to slaves. */
    #define CLIENT_PREVENT_PROP (CLIENT_PREVENT_AOF_PROP|CLIENT_PREVENT_REPL_PROP)
    #define CLIENT_PENDING_WRITE (1<<21) /* Client has output to send but a write
                                            handler is yet not installed. */
    #define CLIENT_REPLY_OFF (1<<22)   /* Don't send replies to client. */
    #define CLIENT_REPLY_SKIP_NEXT (1<<23)  /* Set CLIENT_REPLY_SKIP for next cmd */
    #define CLIENT_REPLY_SKIP (1<<24)  /* Don't send just this reply. */
    #define CLIENT_LUA_DEBUG (1<<25)  /* Run EVAL in debug mode. */
    #define CLIENT_LUA_DEBUG_SYNC (1<<26)  /* EVAL debugging without fork() */ 
