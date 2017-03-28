## 时间事件

> #define run_with_period(_ms_) if ((_ms_ <= 1000/server.hz) || !(server.cronloops%((_ms_)/(1000/server.hz))))
首先我们来解读一些这个宏，server.hz为server默认执行频率。server.cronloops在server初始化
的时候已经初始化为0， 然后在每次执行serverCron的时候做累加操作，所以上面的宏可以理解为
如果ms小于服务器默认执行周期则执行，如果大的话，则等待相应周期之后执行。


