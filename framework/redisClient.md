## redisClient的实现

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

