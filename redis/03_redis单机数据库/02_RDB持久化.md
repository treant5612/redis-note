### RDB持久化

由于Redis是内存数据库，所以为了能在退出后恢复状态，Redis提供了RDB持久化功能，将redis在内存中的状态保存到一个压缩的二进制RDB文件中。



#### RDB文件的创建和载入

有两个命令可以用于生成RDB文件：**SAVE**和**BGSAVE**

- SAVE命令会阻塞Redis服务器进程，直到RDB文件创建完毕，在此期间redisServer无法处理任何请求。
- BGSAVE会派生一个子进程，有子进程负责创建RDB文件，父进程(服务器进程)继续处理命令请求。



#### 持久化时的服务器状态

##### **SAVE命令执行时：**

SAVE命令会阻塞服务器，客户端的所有命令都会被拒绝。只有当SAVE命令执行完毕，重新接受命令请求之后，客户端发送的命令才会被处理

##### **BGSAVE命令执行时的服务器状态：**

BGSAVE命令是由子进程执行的，服务器可以继续处理客户端请求，但是SAVE/BGSAVE命令会被拒绝（竞态），而BGREWRITEAOF会被延后到BGSAVE执行完毕后执行。

> 执行其它命令没有竞态：fork子进程时，内核将代码段标记为只读，对其余内存使用写实复制的计数，即开始时父子进程均指向同样的内存页，之后内核捕获父进程或者子进程对内存页面的修改，为要修改的内存页创建拷贝给相应的进程，所以父子进程的内存数据是相互独立不影响的。即此处BGSAVE保存的是开始执行该命令时redisDb的快照，执行过程间对数据库的操作不会影响RDB文件。

##### **载入RDB文件时：**

服务器会一直阻塞直到载入完成。



#### 自动间隔性保存

可以对服务器的save选项设置多个条件，满足其一时即触发BGSAVE

``` 
save 900 1 //服务器在900s之内至少进行了一次修改
save 300 10 //服务器在300s内至少进行了10次修改
save 60 10000 //服务器在60s内至少进行了10000次修改
```

这个配置是redis的默认设置。在启动服务时，会被载入到redisServer结构体的saveparams数组中。

```c
struct saveparam {
    time_t seconds;
    int changes;
};
-----------------------
struct redisServer{
  	...
    saveparam* saveparams
}
```

**dirty计数器与lastsave**

- **dirty**计数器：上次执行rdbSave后，进行了多少次修改操作(增删改)。

- **lastsave**时间戳，记录上次执行rdbSave的时间

  > rdb.c/rdbSave函数尾部代码包括：将dirty置0，将lastsave置为当前时间

**周期性执行**

redis服务器的周期性函数**serverCron**（redis.c，新版为server.c）默认100ms执行一次(10hz)。其中与持久化相关的代码如下：

``` c

    // 检查 BGSAVE 或者 BGREWRITEAOF 是否已经执行完毕
    if (server.rdb_child_pid != -1 || server.aof_child_pid != -1) {
        int statloc;
        pid_t pid;
        // 接收子进程发来的信号，非阻塞
        if ((pid = wait3(&statloc,WNOHANG,NULL)) != 0) {
            int exitcode = WEXITSTATUS(statloc);
            int bysignal = 0;
            
            if (WIFSIGNALED(statloc)) bysignal = WTERMSIG(statloc);

            // BGSAVE 执行完毕
            if (pid == server.rdb_child_pid) {
                backgroundSaveDoneHandler(exitcode,bysignal);

            // BGREWRITEAOF 执行完毕
            } else if (pid == server.aof_child_pid) {
                backgroundRewriteDoneHandler(exitcode,bysignal);

            } else {
                redisLog(REDIS_WARNING,
                    "Warning, detected child with unmatched pid: %ld",
                    (long)pid);
            }
            updateDictResizePolicy();
        }
    } else {
        // 既然没有 BGSAVE 或者 BGREWRITEAOF 在执行，那么检查是否需要执行它们

        // 遍历所有保存条件，看是否需要执行 BGSAVE 命令
         for (j = 0; j < server.saveparamslen; j++) {
            struct saveparam *sp = server.saveparams+j;
            // 检查是否有某个保存条件已经满足了
            if (server.dirty >= sp->changes &&
                server.unixtime-server.lastsave > sp->seconds &&
                (server.unixtime-server.lastbgsave_try >
                 REDIS_BGSAVE_RETRY_DELAY ||
                 server.lastbgsave_status == REDIS_OK))
            {
                redisLog(REDIS_NOTICE,"%d changes in %d seconds. Saving...",
                    sp->changes, (int)sp->seconds);
                // 执行 BGSAVE
                rdbSaveBackground(server.rdb_filename);
                break;
            }
         }

        // 触发 BGREWRITEAOF
         if (server.rdb_child_pid == -1 &&
             server.aof_child_pid == -1 &&
             server.aof_rewrite_perc &&
             // AOF 文件的当前大小大于执行 BGREWRITEAOF 所需的最小大小
             server.aof_current_size > server.aof_rewrite_min_size)
         {
            // 上一次完成 AOF 写入之后，AOF 文件的大小
            long long base = server.aof_rewrite_base_size ?
                            server.aof_rewrite_base_size : 1;

            // AOF 文件当前的体积相对于 base 的体积的百分比
            long long growth = (server.aof_current_size*100/base) - 100;

            // 如果增长体积的百分比超过了 growth ，那么执行 BGREWRITEAOF
            if (growth >= server.aof_rewrite_perc) {
                redisLog(REDIS_NOTICE,"Starting automatic rewriting of AOF on %lld%% growth",growth);
                // 执行 BGREWRITEAOF
                rewriteAppendOnlyFileBackground();
            }
         }
    }
```





#### RDB文件结构

 RDB文件结构包含以下几个部分

``` 
REDIS|db_version|database|EOF|check_sum
```

- 最前面的是字符串REDIS，长度为5字节。

- db_version长度4字节，保存版本号，如0006代表RDB文件的第六版。(本处只介绍第六版文件结构)

- database保存数据库状态。如果服务器所有数据库都为空，这部分长度为0。
- EOF长度1字节，保存EOF，标志着正文内容结束。
- check_sum 由上述4部分的内容计算得出。

**database部分**

```
SELECTDB (常量，1字节)
db_number(1/2/5字节，内容为数据库编号)
key_value_pairs => [EXPIRETIME_MS|ms|TYPE|key|value] （大写为固定常量，小写为值）
	其中value会根据对象类型、长度、压缩设置来以相应的编码保存。比如无压缩string保存为[len|string]

```

```bash
$ od -c dump.rdb
0000000   R   E   D   I   S   0   0   0   9 372  \t   r   e   d   i   s
0000020   -   v   e   r 005   5   .   0   .   7 372  \n   r   e   d   i
0000040   s   -   b   i   t   s 300   @ 372 005   c   t   i   m   e 302
0000060 361   9   U   ^ 372  \b   u   s   e   d   -   m   e   m 302 270
0000100   i  \r  \0 372  \f   a   o   f   -   p   r   e   a   m   b   l
0000120   e 300  \0 376  \0 373 001  \0  \0 003   M   S   G 005   H   E
0000140   L   L   O 377   ! 003 177 316 217 030 371 340
0000154
//redis5.0的rdb文件结构有所变化，string对象和key的保存方式还是一样的 [len]MSG[len]HELLO
```

