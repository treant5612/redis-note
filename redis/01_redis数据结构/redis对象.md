## redis对象

**redis 对象结构体的定义在redis.h中，实现在object.c中**

//新版本redis是server.h文件定义了这些结构

所有的redis对象，包括string/list/set/zset/hash都是由该结构体所表示。

对于redis数据库所保存的键值对而言，**键总是一个字符串对象**，而值可以是字符串/list/set/zset/hash其中的一种。

#### redis对象类型

redis对象有5种类型，由redisObject结构体中的type定义。type的可能值如下：

```c
#define REDIS_STRING 0
#define REDIS_LIST 1
#define REDIS_SET 2
#define REDIS_ZSET 3
#define REDIS_HASH 4
```

#### redis对象编码

每种类型的对象都至少使用了2种不同的编码，redis对象编码如下

```c
#define REDIS_ENCODING_RAW 0     /* Raw representation */  // 简单动态字符串
#define REDIS_ENCODING_INT 1     /* Encoded as integer */  // long类型的整数
#define REDIS_ENCODING_HT 2      /* Encoded as hash table */// 字典 
#define REDIS_ENCODING_ZIPMAP 3  /* Encoded as zipmap */	// 压缩map
#define REDIS_ENCODING_LINKEDLIST 4 /* Encoded as regular linked list */ //链表
#define REDIS_ENCODING_ZIPLIST 5 /* Encoded as ziplist */	//压缩列表
#define REDIS_ENCODING_INTSET 6  /* Encoded as intset */	//整数集合
#define REDIS_ENCODING_SKIPLIST 7  /* Encoded as skiplist */	//跳跃表
#define REDIS_ENCODING_EMBSTR 8  /* Embedded sds string encoding */	//embstr编码字符串

#define OBJ_ENCODING_QUICKLIST 9 /* Encoded as linked list of ziplists */ //3.2新增

#define OBJ_ENCODING_STREAM 10 /* Encoded as a radix tree of listpacks */// 5.0新增

```

redis对象的编码是随存储的数据而变化的。例如使用sadd nums 1 2 3 初始化一个key为nums的set，其encoding为intset，往其中加入一个字符串（sadd nums "zero"）之后其encoding将变为hashtable



```c
typedef struct redisObject {

    // 类型 见上述对象类型的定义
    unsigned type:4;    

    // 编码
    unsigned encoding:4;

    // 对象最后一次被访问的时间
    unsigned lru:REDIS_LRU_BITS; /* lru time (relative to server.lruclock) */

    // 引用计数
    int refcount;

    // 指向实际值的指针
    void *ptr;

} robj;

```

