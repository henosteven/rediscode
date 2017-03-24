## redis事务的实现

` 尽管现在redis事务用的比较少，或者说是跟lua相比没有任何优势，这里还是要分析一下

    首先当我们执行 multi 命令的时候，client会被标记为CLIENT_MULTI
    c->flags |= CLIENT_MULTI;
    注意：multi不能嵌套

    之后客户端命令会有如下的处理（代码摘自server.c call()方法）
    当客户端的状态处于事物状态的时候，如果当前发送的命令不是exec|discard|multi|watch
    直接放入队列中
    if (c->flags & CLIENT_MULTI &&
            c->cmd->proc != execCommand && c->cmd->proc != discardCommand &&
            c->cmd->proc != multiCommand && c->cmd->proc != watchCommand)
    {
        queueMultiCommand(c); //将当前命令及其参数放入client的mstate中
        addReply(c,shared.queued);
    }...

    记录命令的队列数据结构如下
    typedef struct multiCmd {
        robj **argv; //参数
        int argc; //参数个数
        struct redisCommand *cmd; //对应的命令
    } multiCmd;

    typedef struct multiState {
        multiCmd *commands;     /* Array of MULTI commands */
        int count;              /* Total number of MULTI commands */
        int minreplicas;        /* MINREPLICAS for synchronous replication */
        time_t minreplicas_timeout; /* MINREPLICAS timeout as unixtime. */
    } multiState;     

执行exec命令

* 检查当前状态是否为CLIENT_MULTI， 如果不是直接返回EXEC without MULTI
* 检查当前状态是否为CLIENT_DIRTY_CAS， CLIENT_DIRTY_EXEC，如果命中直接返回然后取消当前事物
    CLIENT_DIRTY_CAS  当前客户端watch的key被修改了
    CLIENT_DIRTY_EXEC 当前执行的命令不存在、参数不正确、或者系统内存不足等情况
* unwatchAllKeys，把当前所有watch的key删除
* 遍历执行cmd，如果涉及到修改数据，需要将multi命令exec均同步到固化文件和从服务器
* handle_monitor @todo

执行discard命令

* freeClientMultiState
* initClientMultiState
* c->flags &= ~(CLIENT_MULTI|CLIENT_DIRTY_CAS|CLIENT_DIRTY_EXEC)
* unwatchAllKeys 
