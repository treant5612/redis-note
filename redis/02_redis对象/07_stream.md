### Stream

Stream(流)是redis5.0中新增的数据结构。用于在redis中实现消息队列。

其基本元素结构如下

> ``` 
> id: timestamp-seq
> kvs: key1:val1,key2,val2
> ```

redis将这些元素储存在radix树(压缩trie树)中。



#### 数据结构

##### id

``` c
//Stream元素id：一个由毫秒时间戳和序列号组合而成的128bit整数。在同一个毫秒内（或者在发生时钟倒退时的一个过去的时间）产生的ID将使用同一个毫秒数，而seq序列号是递增的。
typedef struct streamID {
    uint64_t ms;        /* ms级的unix时间戳. */
    uint64_t seq;       /* 序列号 */
} streamID;
```

##### stream结构

```c
typedef struct stream {
    rax *rax;               /* radix树，保存stream的内容. */
    uint64_t length;        /* stream中的元素数量. */
    streamID last_id;       /* 最大id，如果stream为空则是0 */
    rax *cgroups;           /*消费者字典  name->StreamCG */
} stream;

```

##### rax结构

``` c
typedef struct rax {
    raxNode *head;
    uint64_t numele;
    uint64_t numnodes;
} rax;

typedef struct raxNode {
    uint32_t iskey:1;     /* Does this node contain a key? */
    uint32_t isnull:1;    /* Associated value is NULL (don't store it). */
    uint32_t iscompr:1;   /* Node is compressed. */
    uint32_t size:29;     /* Number of children, or compressed string len. */
    unsigned char data[];	
} raxNode;
// 四个uint32_t总共占用32位=4字节，其前3个占用1bit，size字段占用29bit
// data为柔性数组，即在分配空间时，如 node = (*raxNode)malloc(sizeof(raxNode)+size)，比4字节多的部分供data数组使用。
```



#### 命令

##### XADD

对应 t_stream.c/xaddCommand函数。

```c
/* XADD key [MAXLEN [~|=] <count>] <ID or *> [field value] [field value] ... */
void xaddCommand(client *c) {
    ...
        
    /* Append using the low level function and return the ID. */
    if (streamAppendItem(s,c->argv+field_pos,(c->argc-field_pos)/2,
        &id, id_given ? &id : NULL)
        == C_ERR)
    {
        addReplyError(c,"The ID specified in XADD is equal or smaller than the "
                        "target stream top item");
        return;
    }
    ...
}
```

即 XADD最终是调用streamAppendItem函数实现的。

**streamAppendItem函数**

``` c
int streamAppendItem(stream *s, robj **argv, int64_t numfields, streamID *added_id, streamID *use_id) {
    /* 生成新的id：如有传入id使用传入id，如无则生成一个id */
    streamID id;
    if (use_id)
        id = *use_id;
    else
        streamNextID(&s->last_id,&id);
    
 
    /* 检查新的id 是否比已存在的最大id大 ；即使是自动生成的id也可能因为seq溢出(1ms内uint64溢出？) */
    if (streamCompareID(&id,&s->last_id) <= 0) return C_ERR;
    
    
    /* 添加新的项. */
    raxIterator ri;
    raxStart(&ri,s->rax);		//初始化迭代器，之后必须紧跟raxSeek调用
    raxSeek(&ri,"$",NULL,0);	//$参数代表seek的目标是last，即最后一项
    
    size_t lp_bytes = 0;        /* Total bytes in the tail listpack. */
    unsigned char *lp = NULL;   /* Tail listpack pointer. */
	
    /* 获取stream中最后一个节点的listpack */
    if (raxNext(&ri)) {
        lp = ri.data;
        /*lpBytes调用了lpGetTotalBytes这个宏函数
         *其返回值为 p[0]|p[1]<<8|p[2]<<16|p[3]<<24 
         *即lp指针对应的数据长度        */
        lp_bytes = lpBytes(lp);			
    }
    /* 释放迭代器 */
    raxStop(&ri);
    
    /*将新的key添加到按照字典顺序添加到radix树
     *所以需要将timestamp-seq作为一个128bit大端序的整数
     *来让大多数节点的前缀部分相同*/
    uint64_t rax_key[2];    /* Key in the radix tree containing the listpack.*/
    streamID master_id;     /* ID of the master entry in the listpack. */
    
     /* First of all, check if we can append to the current macro node or
     * if we need to switch to the next one. 'lp' will be set to NULL if
     * the current node is full. */
    if (lp != NULL) {
        if (server.stream_node_max_bytes &&
            lp_bytes >= server.stream_node_max_bytes)
        {
            lp = NULL;
        } else if (server.stream_node_max_entries) {
            int64_t count = lpGetInteger(lpFirst(lp));
            if (count >= server.stream_node_max_entries) lp = NULL;
        }
    }


```

