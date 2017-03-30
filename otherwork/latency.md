## latency的实现

> latency 的基本原理就是，记录开始时间，记录结束时间，然后计算时差，如果
时差在超出了服务器设定阈值，记录该条数据

    涉及到的数据结构如下
    struct latencySample {
        int32_t time; /* 延迟发生的时间 We don't use time_t to force 4 bytes usage everywhere.*/
        uint32_t latency; /* 延迟的时间 Latency in milliseconds. */
    };

    struct latencyTimeSeries {
        int idx; /* 下一个元素的存放位置 Index of the next sample to store. */
        uint32_t max; /* 改事件的最大延迟值 Max latency observed for this event. */
        struct latencySample samples[LATENCY_TS_LEN]; /* 延迟记录存放列表 Latest history. */
    };

    在实际的使用过程中，我们可以看见这样的例子
    mstime_t latency;
    latencyStartMonitor(latency);
    .....
    latencyEndMonitor(latency);
    latencyAddSampleIfNeeded("eviction-cycle",latency);
    这就是一个完整的调用例子，eviction-cycle 是事件的标记对应一个lantencyTimeSeries
    整体存放在server.latency_events

> 整体的数据已经记录好了，分析是件比较清晰的事情
