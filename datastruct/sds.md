## sds的实现
sds是redis高级数据结构string的基础

首先来看看sds的数据结构

	struct __attribute__ ((__packed__)) sdshdr5 {
        unsigned char flags; /* 3 lsb of type, and 5 msb of string length */
        char buf[];
    };
    
    struct __attribute__ ((__packed__)) sdshdr8 {
        uint8_t len; /* used */
        uint8_t alloc; /* excluding the header and null terminator */
        unsigned char flags; /* 3 lsb of type, 5 unused bits */
        char buf[];
    };
    
    struct __attribute__ ((__packed__)) sdshdr16 {
        uint16_t len; /* used */
        uint16_t alloc; /* excluding the header and null terminator */
        unsigned char flags; /* 3 lsb of type, 5 unused bits */
        char buf[];
    };
    
    struct __attribute__ ((__packed__)) sdshdr32 {
        uint32_t len; /* used */
        uint32_t alloc; /* excluding the header and null terminator */
        unsigned char flags; /* 3 lsb of type, 5 unused bits */
        char buf[];
    };
    
    struct __attribute__ ((__packed__)) sdshdr64 {
        uint64_t len; /* used */
        uint64_t alloc; /* excluding the header and null terminator */
        unsigned char flags; /* 3 lsb of type, 5 unused bits */
        char buf[];
    };


从上面的数据结构中我们可以看出，针对不同长度的字符串，redis使用的数据结构是不一样，这样做的主要目的是节约内存使用。

	len   : 字符串的长度（保证了字符串的二进制安全）
	
	alloc : 不包括header在内的已申请内存空间（字符串变化的时候可以确认申请的内存是否充足）
	
	flags : 标记
	
	buf   : 字符串的存放位置（弹性数组）

`尤其需要注意的一点就是__attribute__ ((__packed__))的使用，保证了兼容性，方便获取flags`

1、在使用过程中，对于给定的字符串，我们的首先需要根据字符串的长度来选择不同的结构体来表示。

2、typedef char *sds;  表示sds可以出现在char出现的任何位置，因为sds使用的时候都是指针实在buf的位置，如果需要获取len的话，首先需要使用s[-1]获取flags，然后根据flags来确定(s - sizeof(header))->len

3、新建sds的时候，会根据字符串的长度选择不同的数据类型存放，然后申请内存。

` 重要的空间申请方法 `

    sds sdsMakeRoomFor(sds s, size_t addlen) {
        void *sh, *newsh;
        
        /* 获取我们字符串的剩余空间 */
        size_t avail = sdsavail(s);
        
        size_t len, newlen;
        char type, oldtype = s[-1] & SDS_TYPE_MASK;
        int hdrlen;

        /* Return ASAP if there is enough space left.
         * ASAP : as soon as possible
         * 如果我们的剩余空间充足的话，直接返回不用操作了
         */
        if (avail >= addlen) return s;

        /*
         * #define SDS_MAX_PREALLOC (1024*1024)
         *
         * 获取原有长度
         * 新长度 = 原有长度 + 新增长度
         * 如果新长度 < 1MB =》 新长度 *= 2
         *          >= 1MB =》 新长度 += 1M
         * 也是就是说，如果s是1M的话，实际要求的空间为2MB, 2MB =》 3MB
         * 如果新空间是 300KB的话，实际要求的空间是600KB
         */
        len = sdslen(s);
        sh = (char*)s-sdsHdrSize(oldtype);
        newlen = (len+addlen);
        if (newlen < SDS_MAX_PREALLOC)
            newlen *= 2;
        else
            newlen += SDS_MAX_PREALLOC;

        /* 因为长度变化，所以要求的类型也不一样了 */
        type = sdsReqType(newlen);

        /* Don't use type 5: the user is appending to the string and type 5 is
         * not able to remember empty space, so sdsMakeRoomFor() must be called
         * at every appending operation.
         * 因为类型为5的flag => 前三位为flag, 后五位为长度，但是到底使用了多少是不知道
         */
        if (type == SDS_TYPE_5) type = SDS_TYPE_8;

        hdrlen = sdsHdrSize(type);

        /* 类型没变的话，直接realloc内存
         * 否则需要申请一块新内存，然后执行copy操作，之后在重置一下len、alloc等
         */
        if (oldtype == type) {
            newsh = s_realloc(sh, hdrlen+newlen+1);
            if (newsh == NULL) return NULL;
            s = (char*)newsh+hdrlen;
        } else {
            /* Since the header size changes, need to move the string forward,
             * and can't use realloc *
            newsh = s_malloc(hdrlen+newlen+1);
            if (newsh == NULL) return NULL;
            memcpy((char*)newsh+hdrlen, s, len+1);
            s_free(sh);
            s = (char*)newsh+hdrlen;
            s[-1] = type;
            sdssetlen(s, len);
        }
        sdssetalloc(s, newlen);
        return s;
    }
