## redisObject

> redisObject是贯穿redis的数据结构，可以这么说是所有redis数据结构的最外层的封装，string | list | set

	数据结构
	typedef struct redisObject {
    	unsigned type:4;
    	unsigned encoding:4;
    	unsigned lru:LRU_BITS; /* lru time (relative to server.lruclock) */
    	int refcount;
    	void *ptr;
	} robj;
	
	
	/* Object types 对象类型分为以下几种*/
	OBJ_STRING 0
	OBJ_LIST 1
	OBJ_SET 2
	OBJ_ZSET 3
	OBJ_HASH 4
	
 	对象的编码类型，例如同样是strings类型的数据，有可能是OBJ_ENCODING_RAW 如果全是数字的话，有可能是OBJ_ENCODING_INT。所以type和encoding共同决定ptr的解析方式
 	
	#define OBJ_ENCODING_RAW 0     /* Raw representation */
	#define OBJ_ENCODING_INT 1     /* Encoded as integer */
	#define OBJ_ENCODING_HT 2      /* Encoded as hash table */
	#define OBJ_ENCODING_ZIPMAP 3  /* Encoded as zipmap */
	#define OBJ_ENCODING_LINKEDLIST 4 /* Encoded as regular linked list */
	#define OBJ_ENCODING_ZIPLIST 5 /* Encoded as ziplist */
	#define OBJ_ENCODING_INTSET 6  /* Encoded as intset */
	#define OBJ_ENCODING_SKIPLIST 7  /* Encoded as skiplist */
	#define OBJ_ENCODING_EMBSTR 8  /* Embedded sds string encoding */
	#define OBJ_ENCODING_QUICKLIST 9 /* Encoded as linked list of ziplists *
	

	lru需要认真分析一下
	

### redisObject的使用
> 引用一段db.c中的代码使用解释，参数中key和value都是robj类型, 这个setKey在我们执行set命令的时候被调用，也就是说key和value都被redisObject封装了。

	void setKey(redisDb *db, robj *key, robj *val) {
    	if (lookupKeyWrite(db,key) == NULL) {
        	dbAdd(db,key,val);
   		} else {
        	dbOverwrite(db,key,val);
    	}
    	incrRefCount(val);
    	
    	/*
    	 * 将key从db->expires中删除，设置了新值，之前存在的过期时间就要被删除
    	 */
    	removeExpire(db,key);
    	
    	/* 每次的对数据库有修改的操作都会调用这个方法
    	 * 这个方法的主要目的是告诉所有watch了这个key的clients，这个值已经被调整了
    	 * 设置标记CLIENT_DIRTY_CAS（check and set）
    	 */
    	signalModifiedKey(db,key); 
	}

	
	
