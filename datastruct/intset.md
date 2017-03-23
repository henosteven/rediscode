
## intset的实现

> intset主要使用在set场景，当元素全部是数字的时候，直接可以使用intset表示，当出现字符串的时候会整体转换为dict。

----



> 首先我们看intset的数据结构
> 其中的encoding类型分为：	
> 
 * INTSET_ENC_INT16 
 * INTSET_ENC_INT32 
 * INTSET_ENC_INT64

	typedef struct intset {
    	uint32_t encoding; //当前编码类型
    	uint32_t length; //元素个数
    	int8_t contents[];
	} intset;
	
	从上面的数据结构可以看出，redis根据元素本身的大小来决定encoding，这样做的主要目的就是充分节约内存的使用。
	encoding的选择代码如下：
	
	static uint8_t _intsetValueEncoding(int64_t v) {
    	if (v < INT32_MIN || v > INT32_MAX)
        	return INTSET_ENC_INT64;
    	else if (v < INT16_MIN || v > INT16_MAX)
        	return INTSET_ENC_INT32;
    	else
        	return INTSET_ENC_INT16;
	}

----

新建intset
	
	新建一个intset,默认为选择encoding为INTSET_ENC_INT16
	
	intset *intsetNew(void) {
    	intset *is = zmalloc(sizeof(intset));
    	is->encoding = intrev32ifbe(INTSET_ENC_INT16);
    	is->length = 0;
    	return is;
	}

----

插入元素
	
	插入元素的时候有一点需要注意，如果当前encoding比新插入的元素的encoding小， 则需要整体更新现有intset，否则直接搜索插入位置然后插入元素（如果元素已经存在，则不插入，直接返回）	
	更新intset的时候，采用的方式是，先扩容，然后从最后一个元素开始扩展。
	
----

搜索元素
	
	有一个需要可以提出的地方，因为有encoding的关系，所以可以直接对比encoding，如果搜索元素比当前的encoding大，则搜索失败，否则直接二分法查找
	




