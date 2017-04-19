## slowlog的实现

> slowlog 的基本原理就是，记录命令执行开始时间，依次计算命令执行时间，如果
时差在超出了服务器设定阈值(默认为CONFIG_DEFAULT_SLOWLOG_LOG_SLOWER_THAN 10000)，记录该条数据到server.slowlog

    涉及到的数据结构如下
    typedef struct slowlogEntry {
        robj **argv;
        int argc;
        long long id;       /* Unique entry identifier. */
        long long duration; /* Time spent by the query, in nanoseconds. */
        time_t time;        /* Unix time at which the query was executed. */
    } slowlogEntry;

    在实际的使用过程中，我们可以看见这样的例子
    start = ustime();
    c->cmd->proc(c);

    /* 计算运行时间
     *
     * 计算运行时间 -》latencyAddSampleIfNeeded 添加到延迟监控中
     *              -》slowlogPushEntryIfNeeded 添加到慢查询中
     */
    duration = ustime()-start;
    if (flags & CMD_CALL_SLOWLOG && c->cmd->proc != execCommand) {
        char *latency_event = (c->cmd->flags & CMD_FAST) ?
            "fast-command" : "command";
        latencyAddSampleIfNeeded(latency_event,duration/1000);
        slowlogPushEntryIfNeeded(c->argv,c->argc,duration);
    }

> 整体的数据已经记录好了，然后调用命令slowlog就可以显示数据了
