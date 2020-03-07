### ziplist



#### 新建压缩列表

unsigned char *ziplistNew(void) 

默认分配的内存大小是 sizeof(uint32_t) * 2 +sizeof(uint16) + 1

其结构为:

```chinese
-----------------------------------------------
zlbytes | zltail | zllen | entries... | zlend |
-----------------------------------------------
zlbytes: 4字节，记录压缩列表大小
zltail: 4字节，记录压缩列表表尾偏移量
zllen: 2字节，压缩列表节点数量（当大小超过2字节范围时需要遍历才能得到真实大小）
entries 数量不定，每个节点的长度也不确定
zlend: 压缩列表末尾标记 值0xFF
```

所以新建的压缩列表大小是  zlbytes(4) + zltail(4) + zllen(2) + zlend(1) 

#### 压缩列表节点

```
----------------------------------------------
| previous_entry_length | encoding | content | 
----------------------------------------------

previous_entry_length: 上一个节点的长度，如上个节点长度小于254，则占用1字节，否则其长度为5：1个字节为0xFF，后续4字节保存实际长度
【这个值的用途是：通过当前节点位置和上个节点长度，可以直接计算得到上个节点的位置。(比如从表尾向表头遍历)】

encoding： 可能是 1字节/2字节/5字节 长。最高位为00，01，10的是字节数组编码，去掉最高两位的其它位记录字节数组长度。 如果最高位位11，则长度为1字节，content为整数，其类型和长度由剩余6bit记录（如11000000表示content为int16_t类型的整数，而1111xxxx表示无content属性，由xxxx保存一个0~15之间的值）。

content: 保存节点的值，其长度和类型由encoding决定。

```

####  连锁更新

每个节点都保存前一个节点的长度。假如有e0~en一个n+1个节点长度为250~253，那么记录这些节点只需要1字节的previous_content_length属性。假如在e0之前插入一个长度大于254的新节点，那么e0的长度(previous+encoding+content) 将会由于previous从1字节变为5字节而增加，从而会导致后续的所有节点都被扩展。【连锁更新的最坏复杂度为N^2，但是实际中恰好存在n个250~253长度的节点的概率很小，所以通常认为ziplist的平均时间复杂度是o(N)的】

---

ziplist.h头文件中定义了压缩列表相关的函数

``` c
#define ZIPLIST_HEAD 0
#define ZIPLIST_TAIL 1

unsigned char *ziplistNew(void);
unsigned char *ziplistPush(unsigned char *zl, unsigned char *s, unsigned int slen, int where);
unsigned char *ziplistIndex(unsigned char *zl, int index);
unsigned char *ziplistNext(unsigned char *zl, unsigned char *p);
unsigned char *ziplistPrev(unsigned char *zl, unsigned char *p);
unsigned int ziplistGet(unsigned char *p, unsigned char **sval, unsigned int *slen, long long *lval);
unsigned char *ziplistInsert(unsigned char *zl, unsigned char *p, unsigned char *s, unsigned int slen);
unsigned char *ziplistDelete(unsigned char *zl, unsigned char **p);
unsigned char *ziplistDeleteRange(unsigned char *zl, unsigned int index, unsigned int num);
unsigned int ziplistCompare(unsigned char *p, unsigned char *s, unsigned int slen);
unsigned char *ziplistFind(unsigned char *p, unsigned char *vstr, unsigned int vlen, unsigned int skip);
unsigned int ziplistLen(unsigned char *zl);
size_t ziplistBlobLen(unsigned char *zl);
```

