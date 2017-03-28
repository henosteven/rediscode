## pubsub的实现

> 整体思路为：客户端关注某个key值（频道），一旦该key值（频道）被传入信息（publish）
就通知该客户端，也就是订阅接收的概念。具体实现依赖server.pubsub_channels和server.pubsub_patterns。 

当我们执行订阅命令的时候，发生了什么? 核心涉及到两个操作，将频道添加到当前客户端的
订阅列表，将客户端添加到服务端对应订阅了该频道的客户端列表。

    int pubsubSubscribeChannel(client *c, robj *channel) {
        dictEntry *de;
        list *clients = NULL;
        int retval = 0;

        /* Add the channel to the client -> channels hash table 
         * 首先将订阅的频道加入到客户端的pubsub_channels字典里
         */
        if (dictAdd(c->pubsub_channels,channel,NULL) == DICT_OK) {
            retval = 1;
            incrRefCount(channel);

            /* Add the client to the channel -> list of clients hash table 
             * 取出服务端订阅了该频道的所有clientlist
             * 如果之前没有client订阅该频道则新建一个list
             * 如果存在则直接取出
             */
            de = dictFind(server.pubsub_channels,channel);
            if (de == NULL) {
                clients = listCreate();
                dictAdd(server.pubsub_channels,channel,clients);
                incrRefCount(channel);
            } else {
                clients = dictGetVal(de);
            }   
            
            /* 将当前客户端放入server的对应该频道的client list队尾 */
            listAddNodeTail(clients,c);
        }   
        /* Notify the client */
        addReply(c,shared.mbulkhdr[3]);
        addReply(c,shared.subscribebulk);
        addReplyBulk(c,channel);
        addReplyLongLong(c,clientSubscriptionsCount(c));
        return retval;
    }

当我们执行publish的时候发生了什么？核心分为两部分，取出上面服务端保存的定了某个频道的
客户端list，然后发送消息。取出

    int pubsubPublishMessage(robj *channel, robj *message) {
        int receivers = 0;
        dictEntry *de;
        listNode *ln;
        listIter li;

        /* Send to clients listening for that channel */
        /* 获取订阅该频道的客户端列表 */
        de = dictFind(server.pubsub_channels,channel);
        if (de) {
            list *list = dictGetVal(de);
            listNode *ln;
            listIter li;

            listRewind(list,&li);
            while ((ln = listNext(&li)) != NULL) {
                client *c = ln->value;
                
                /* 发送消息 */
                addReply(c,shared.mbulkhdr[3]);
                addReply(c,shared.messagebulk);
                addReplyBulk(c,channel);
                addReplyBulk(c,message);
                receivers++;
            }
        }

        /* Send to clients listening to matching channels
         * 遍历模式列表，查找对应的用户进行消息发送(每个节点都是一个client+模式)
         */
        if (listLength(server.pubsub_patterns)) {
            listRewind(server.pubsub_patterns,&li);
            channel = getDecodedObject(channel);
            while ((ln = listNext(&li)) != NULL) {
                pubsubPattern *pat = ln->value;

                if (stringmatchlen((char*)pat->pattern->ptr,
                                    sdslen(pat->pattern->ptr),
                                    (char*)channel->ptr,
                                    sdslen(channel->ptr),0)) {
                    addReply(pat->client,shared.mbulkhdr[4]);
                    addReply(pat->client,shared.pmessagebulk);
                    addReplyBulk(pat->client,pat->pattern);
                    addReplyBulk(pat->client,channel);
                    addReplyBulk(pat->client,message);
                    receivers++;
                }
            }
            decrRefCount(channel);
        }
        return receivers;
    }

    //上面说的server.pubsub_patterns对应的node类型
    typedef struct pubsubPattern {
        client *client;
        robj *pattern;
    } pubsubPattern;
