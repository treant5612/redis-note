## sds的定义:sds.h

```c
//类型别名
typedef char *sds;

struct sdshdr {
    
    // buf 中已占用空间的长度
    int len;

    // buf 中剩余可用空间的长度
    int free;

    // 数据空间
    char buf[];
};
```

与c语言中的字符串相比，sds使用两个int变量来记录字符串的长度和剩余空间。

buf指向实际数据空间的地址。保存的字符串遵循c语言惯例以**'\0'**结尾，但是这1字节不计入len，对于sds而言这个空字符是完全透明的。其目的是便于使用c语言中的字符串函数库中的函数。

#### 相比于char*优势

- **常数复杂度获取字符串长度**

- **杜绝缓冲区溢出（指针访问越界）**

- **减少修改时的内存重分配**

  c中字符串总是N+1的长度，每次修改都需要重新分配空间。

  策略：

  1.空间预分配（小于1m时 分配2n+1的空间，大于1m时额外分配1m的空间）

  2.空间惰性回收：修改时不会立即释放buf中空余的空间

- **二进制安全**：由于是以长度标记结尾，所以字符串中可以包含空字符'\0'

- **兼容部分c字符串函数 **



```c
sds sdsnewlen(const void *init, size_t initlen);
sds sdsnew(const char *init);
sds sdsempty(void);
size_t sdslen(const sds s);
sds sdsdup(const sds s);
void sdsfree(sds s);
size_t sdsavail(const sds s);
sds sdsgrowzero(sds s, size_t len);
sds sdscatlen(sds s, const void *t, size_t len);
sds sdscat(sds s, const char *t);
sds sdscatsds(sds s, const sds t);
sds sdscpylen(sds s, const char *t, size_t len);
sds sdscpy(sds s, const char *t);
sds sdscatfmt(sds s, char const *fmt, ...);
sds sdstrim(sds s, const char *cset);
void sdsrange(sds s, int start, int end);
void sdsupdatelen(sds s);
void sdsclear(sds s);
int sdscmp(const sds s1, const sds s2);
sds *sdssplitlen(const char *s, int len, const char *sep, int seplen, int *count);
void sdsfreesplitres(sds *tokens, int count);
void sdstolower(sds s);
void sdstoupper(sds s);
sds sdsfromlonglong(long long value);
sds sdscatrepr(sds s, const char *p, size_t len);
sds *sdssplitargs(const char *line, int *argc);
sds sdsmapchars(sds s, const char *from, const char *to, size_t setlen);
sds sdsjoin(char **argv, int argc, char *sep);

```

