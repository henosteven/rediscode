## blpop的实现

> 首先摘出两个关键的方法

    void blockingPopGenericCommand(client *c, int where) {
        robj *o;
        mstime_t timeout;
        int j;

        if (getTimeoutFromObjectOrReply(c,c->argv[c->argc-1],&timeout,UNIT_SECONDS)
            != C_OK) return;

        /* 按顺序访问KEY参数 */
        for (j = 1; j < c->argc-1; j++) {
            
            //在db中查找key值对应的数据
            o = lookupKeyWrite(c->db,c->argv[j]);
            if (o != NULL) {
                if (o->type != OBJ_LIST) {
                    //类型判断
                    addReply(c,shared.wrongtypeerr);
                    return;
                } else {
                    if (listTypeLength(o) != 0) {
                        /* Non empty list, this is like a non normal [LR]POP. */
                        char *event = (where == LIST_HEAD) ? "lpop" : "rpop";

                        /* 把元素弹出，这里使用的数据结构是quicklist */
                        robj *value = listTypePop(o,where);
                        serverAssert(value != NULL);

                        addReplyMultiBulkLen(c,2);
                        addReplyBulk(c,c->argv[j]);
                        addReplyBulk(c,value);
                        decrRefCount(value);

                        /* 这里还做了一个操作就是往将空间发一个消息 */
                        notifyKeyspaceEvent(NOTIFY_LIST,event,
                                            c->argv[j],c->db->id);

                        /* 弹出来要是空了的话，直接从db中删除该项
                         * 一头压入，一头消费，应该是会经常空才对，删除了岂不是尴尬了
                         */
                        if (listTypeLength(o) == 0) {
                            dbDelete(c->db,c->argv[j]);
                            notifyKeyspaceEvent(NOTIFY_GENERIC,"del",
                                                c->argv[j],c->db->id);
                        }
                        signalModifiedKey(c->db,c->argv[j]);

                        /* 其实是通知一下从服务器, 删除数据, 但是使用的命令是要调整的，不然就尴尬了塞住了*/
                        server.dirty++;

                        /* Replicate it as an [LR]POP instead of B[LR]POP.
                         * 需要做一个转换
                         */
                        rewriteClientCommandVector(c,2,
                            (where == LIST_HEAD) ? shared.lpop : shared.rpop,
                            c->argv[j]);
                        return;
                    }
                }
            }
        }

        /* If we are inside a MULTI/EXEC and the list is empty the only thing
         * we can do is treating it as a timeout (even with timeout 0).
         *
         * 如果我们当前处于事务模式，如果list是空的话
         * 我们直接把这种情况当做是超时处理
         * 哪怕是你设置的timeout是0也不行
         */
        if (c->flags & CLIENT_MULTI) {
            addReply(c,shared.nullmultibulk);
            return;
        }

        /* If the list is empty or the key does not exists we must block */
        blockForKeys(c, c->argv + 1, c->argc - 2, timeout, NULL);
    }


    /* Set a client in blocking mode for the specified key, with the specified
     * timeout
     * blockForKeys(c, c->argv + 1, c->argc - 2, timeout, NULL);
     *
     * 该方法的主逻辑就是：
     * 遍历所有传入的key, 首先把key添加到自己的blockState里
     * 然后把自己添加到db的中key => clientlist 对应的列表里，如果没有的该列表的话就创建一个
     * 最后将客户端标记为block的客户端，然后主服务block的client+1
     */
    void blockForKeys(client *c, robj **keys, int numkeys, mstime_t timeout, robj *target) {
        dictEntry *de;
        list *l;
        int j;

        c->bpop.timeout = timeout;
        c->bpop.target = target;

        if (target != NULL) incrRefCount(target);

        for (j = 0; j < numkeys; j++) {
            /* If the key already exists in the dict ignore it. 
             * 如果当前key已经在bpop数据中，直接跳过
             */
            if (dictAdd(c->bpop.keys,keys[j],NULL) != DICT_OK) continue;
            incrRefCount(keys[j]);

            /* And in the other "side", to map keys -> clients */
            de = dictFind(c->db->blocking_keys,keys[j]);
            if (de == NULL) {
                int retval;

                /* For every key we take a list of clients blocked for it */
                l = listCreate();
                retval = dictAdd(c->db->blocking_keys,keys[j],l);
                incrRefCount(keys[j]);
                serverAssertWithInfo(c,keys[j],retval == DICT_OK);
            } else {
                l = dictGetVal(de);
            }

            //将当前client添加到blockkey的队尾
            listAddNodeTail(l,c);
        }
        blockClient(c,BLOCKED_LIST);
    }

涉及到核心操作

    server.c processCommand函数中

    if (listLength(server.ready_keys))
            handleClientsBlockedOnLists(); //遍历所有ready_keys然后处理key对应的client

    -------

    每次dbAdd的时候，如果类型是list都会有如下操作

    void dbAdd(redisDb *db, robj *key, robj *val) {
        sds copy = sdsdup(key->ptr);
        int retval = dictAdd(db->dict, copy, val);

        serverAssertWithInfo(NULL,key,retval == DICT_OK);

        /* 核心的一句话来了，之前一直是忽略状态哈哈
         * 😁
         * 如果是队列类型元素的话，需要处理一block的问题
         * 这个操作完成的就是往ready_keys写入相应的key，在命令运行完毕的时候
         * 在主流程中有对这个key的处理，也就是上面的handleClientsBlockedOnLists
         */
        if (val->type == OBJ_LIST) signalListAsReady(db, key);
        if (server.cluster_enabled) slotToKeyAdd(key);
     }

