# redis的时间事件分析

` 时间事件是redis中非常重要的角色，在redis的代码中我们会看见
  aeCreateTimeEvent(server.el, 1, serverCron, NULL, NULL)
  serverCron的主要任务就是执行各类周期性任务

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

