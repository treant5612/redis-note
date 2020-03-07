### 字典



### rehash

#### 

每个字典结构都包含一个长度为2 的哈希表数组，通常使用ht[0]中的数据，而ht[1]为空。在rehash时，会为ht[1]分配相应空间，然后将保存在ht[0]中的所有条目rehash到ht[1]中，当所有的条目都迁移到h[1]时，将释放原来的h[0]并将h[1]设为新的h[0]。

#### 负载因子

哈希表的负载因子 = 键值对数量/哈希表大小(数组长度)

#### 哈希表扩展与收缩

发生条件:

- 服务器没有执行BGSAVE/BGREWRITEAOF命令&& 负载因子大于等于1
- 服务器执行BGSAVE/BGREWRITEAOF命令时&& 负载因子大于等于5
- 收缩：负载因子小于等于0.1

#### 渐进式rehash

dict结构体中 由rehashidx属性。当未进行rehash时，其值为-1。

在对该字典的正常操作过程中，redis会将dictEntry逐个移动到ht[1]中，每移动一个，将rehashidx的值+1。最终所有的值都移动到ht[1]当中时，再将rehashidx重置为-1。

在rehash发生的过程中，redis会在ht[0]上查找，找不到再去ht[1]中查找。而新增的数据则一律保存到ht[1]当中

#### 字典结构体


```c
typedef struct dict {

    // 类型特定函数
    dictType *type;

    // 私有数据
    void *privdata;

    // 哈希表
    dictht ht[2];

    // rehash 索引
    // 当 rehash 不在进行时，值为 -1
    int rehashidx; /* rehashing not in progress if rehashidx == -1 */

    // 目前正在运行的安全迭代器的数量
    int iterators; /* number of iterators currently running */

} dict;

```





#### 字典节点

``` c
typedef struct dictEntry {
    
    // 键
    void *key;

    // 值
    union {
        void *val;
        uint64_t u64;
        int64_t s64;
    } v;

    // 指向下个哈希表节点，形成链表
    struct dictEntry *next;

} dictEntry;
```

#### 字典类型结构体

``` c
typedef struct dictType {

    // 计算哈希值的函数
    unsigned int (*hashFunction)(const void *key);

    // 复制键的函数
    void *(*keyDup)(void *privdata, const void *key);

    // 复制值的函数
    void *(*valDup)(void *privdata, const void *obj);

    // 对比键的函数
    int (*keyCompare)(void *privdata, const void *key1, const void *key2);

    // 销毁键的函数
    void (*keyDestructor)(void *privdata, void *key);
    
    // 销毁值的函数
    void (*valDestructor)(void *privdata, void *obj);

} dictType;
```

#### 字典哈希表结构体

包含一个dictEntry指针 的数组，数组大小，掩码以及总节点数量

这不是字典的最终实现，只是用于保存数据的结构

``` c
typedef struct dictht {
    
    // 哈希表数组
    dictEntry **table;

    // 哈希表大小
    unsigned long size;
    
    // 哈希表大小掩码，用于计算索引值
    // 总是等于 size - 1
    unsigned long sizemask;

    // 该哈希表已有节点的数量
    unsigned long used;

} dictht;

```

