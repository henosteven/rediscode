# redis的时间事件分析

` 时间事件是redis中非常重要的角色，在redis的代码中我们会看见
  aeCreateTimeEvent(server.el, 1, serverCron, NULL, NULL)
  serverCron的主要任务就是执行各类周期性任务
`

> 我们来看一段调用代码

    void aeMain(aeEventLoop *eventLoop) {
        eventLoop->stop = 0;
        while (!eventLoop->stop) {
            if (eventLoop->beforesleep != NULL)
                eventLoop->beforesleep(eventLoop);
            aeProcessEvents(eventLoop, AE_ALL_EVENTS);
        }
    }

> 在aeProcessEvents函数中

    int aeProcessEvents(aeEventLoop *eventLoop, int flags)    
        ......
        if (flags & AE_TIME_EVENTS)
                processed += processTimeEvents(eventLoop);
        ......
    }

> 所以我们可以看出，时间事件的调用在主循环中，但是频率是如何控制的呢？
   
    答案就是我们的 server.hz（赫兹）
    #define run_with_period(_ms_) if ((_ms_ <= 1000/server.hz) || !(server.cronloops%((_ms_)/(1000/server.hz))))
    在serverCron中经常能看见这样的代码
    run_with_period(5000) {
      serverLog(LL_VERBOSE,
              "%lu clients connected (%lu slaves), %zu bytes in use",
              listLength(server.clients)-listLength(server.slaves),
              listLength(server.slaves),
              zmalloc_used_memory());
    }
    通过上面的宏我们可以看见
    如果5000<=1000/server.hz表示事件要求的频率比我们server.hz要大，立刻执行
    否者需要累加到一个周期之后再进行

> serverCron中包含的一些重要操作

* clientCron
* sentinelTimer
