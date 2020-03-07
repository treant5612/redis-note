## redis



### 启动入口

redis的入口在server.c/main函数。redis服务端的状态保存在server.h/redisServer类型的全局变量server中。

其大致的启动流程如下

``` c
int main(int argc, char **argv) {
	... //初始化时间区域，随机数种子等等
        
    initServerConfig(); /*将server变量的参数设置为默认值
     *比如server.hz=10，默认100ms运行依次serverCron
     *在这个函数的末尾会调用initConfigValues，端口等的信息在这个函数中进行设置 */

    ... //模组、哨兵、加载启动参数等等等    
   
    initServer(); /* 设置信号处理器(setupSignalHandlers)
     * 初始化配置，创建各种server内部数据结构如clients链表，db数组等等
     * 创建 server.el(EventLoop)     aeCreateEventLoop;
     * 对server.port指定的端口进行监听  listenToPort;
     * 调用 aeCreateTimeEvent 注册时间事件及其处理函数serverCron
     * 调用 aeCreateFileEvent 注册文件事件处理函数acceptTcpHandler 
    */

    ... //redisAsciiArt(); 哨兵 模组 等等
    
    
    aeSetBeforeSleepProc(server.el,beforeSleep);
    aeSetAfterSleepProc(server.el,afterSleep);
    aeMain(server.el);		//主事件循环
    aeDeleteEventLoop(server.el);

}
```



### 创建事件循环

> aeEventLoop *aeCreateEventLoop(int setsize)

#### 事件循环结构

事件循环的结构定义在ae.h中：

```c
typedef struct aeEventLoop {
    int maxfd;   /* highest file descriptor currently registered */
    int setsize; /* 最大连接数，默认配置为10000，可修改 */
    long long timeEventNextId;	/*用于生成时间事件的id*/
    time_t lastTime;     /* 用于检测系统时钟偏移量 */
    aeFileEvent *events; /* 已注册事件 */
    aeFiredEvent *fired; /* 已就绪事件 */
    aeTimeEvent *timeEventHead;	 /* 时间事件 */
    int stop;
    void *apidata; /* IO多路复用API的私有数据 */
    aeBeforeSleepProc *beforesleep;
    aeBeforeSleepProc *aftersleep;
    int flags;
} aeEventLoop;

```

其中文件事件/已就绪事件/时间事件的结构如下

``` c
/* 文件事件结构 */
typedef struct aeFileEvent {
    int mask; /* one of AE_(READABLE|WRITABLE|BARRIER) */
    aeFileProc *rfileProc;	/* 可读事件处理函数 */
    aeFileProc *wfileProc;  /* 可写事件处理函数 */
    void *clientData;	//todo
} aeFileEvent;

typedef void aeFileProc(struct aeEventLoop *eventLoop, int fd, void *clientData, int mask);
```

``` c
/* 已就绪事件 */
typedef struct aeFiredEvent {
    int fd;
    int mask;		//事件标志 READABLE/WRITABLE
} aeFiredEvent;
```

``` c
/* 时间事件 */
typedef struct aeTimeEvent {
    long long id; /* 时间事件id. */
    long when_sec; /* 秒数 */
    long when_ms; /* 毫秒数 */
    aeTimeProc *timeProc;	/* 事件事件处理器 */
    aeEventFinalizerProc *finalizerProc;	/* todo */
    void *clientData;
    struct aeTimeEvent *prev;
    struct aeTimeEvent *next;
} aeTimeEvent;

typedef int aeTimeProc(struct aeEventLoop *eventLoop, long long id, void *clientData);
typedef void aeEventFinalizerProc(struct aeEventLoop *eventLoop, void *clientData);

```

#### 创建事件循环

在完成eventLoop空间分配，属性设置之后，**aeCreateEventLoop**函数会调用**aeApiCreate**函数来创建对应系统实例。

redis封装了4中不同的底层实现，对应不同的系统环境：epoll，kqueue，evport和select。在编译时，会根据宏来选择对应的文件参与最终编译。在内核>2.5.44的linux系统中最终使用的是epoll。

``` c
//ae_epoll.c/aeApiCreate
static int aeApiCreate(aeEventLoop *eventLoop) {
    /* 分配空间，state结构包含一个整型epoll描述符epfd和一个epoll_event指针 */
    aeApiState *state = zmalloc(sizeof(aeApiState));

    if (!state) return -1;
    /* 根据eventLoop.setsize的大小为state->events分配空间*/
    /* epoll_event大小为8，包含一个int类型[其实是union]文件描述符fd和一个uint32类型的事件类型 */
    state->events = zmalloc(sizeof(struct epoll_event)*eventLoop->setsize);
    if (!state->events) {
        zfree(state);
        return -1;
    }
    /* 系统调用，创建epoll实例并获得其文件描述符 */
    state->epfd = epoll_create(1024); /* 1024 is just a hint for the kernel */
    if (state->epfd == -1) {
        zfree(state->events);
        zfree(state);
        return -1;
    }
    /* apidata是void*类型 */
    eventLoop->apidata = state;
    return 0;
}
```

在这一步中 创建了 eventLoop->apidata->events 这个大小为setsize的数组，其中的event是在sys/epoll.h中定义的epoll事件,在man手册中可以查到( man 2 epoll_ctl )：

```c
typedef union epoll_data {
	void        *ptr;
	int          fd;
	uint32_t     u32;
	uint64_t     u64;
} epoll_data_t;

struct epoll_event {
	uint32_t     events;      /* Epoll events */
	epoll_data_t data;        /* User data variable */
};
// 其中uint32_t events的值为 EPOLLIN,EPOLLOUT等事件类型之一。
```

此前，在aeCreateEventLoop函数中，也创建了一个拥有setsize个元素的events的数组，其元素类型为aeFileEvent，包含事件类型mask（api实现为epoll时，对应redis封装后的EPOLLIN/EPOLLOUT等事件类型）和相应的处理函数。



### 监听端口

监听端口的部分在initServer函数中，在事件循环创建之后，会根据相关设置来监听tcp端口，tls tcp端口或者unix socket。此处只分析监听其tcp端口的部分。

监听tcp端口的函数为 server.c/listenToPort。

listenToPort会检查要绑定的地址，如果未指定，则默认绑定到0.0.0.0，即监听本机所有地址。对于IPv4，会调用anet.c/anetTcpServer函数。在anetTcpServer的实现中，会创建socket文件描述符。之后会调用anetListen函数，在这个函数中会调用bind和listen：

``` c
static int anetListen(char *err, int s, struct sockaddr *sa, socklen_t len, int backlog) {
    if (bind(s,sa,len) == -1) {
        anetSetError(err, "bind: %s", strerror(errno));
        close(s);
        return ANET_ERR;
    }

    if (listen(s, backlog) == -1) {
        anetSetError(err, "listen: %s", strerror(errno));
        close(s);
        return ANET_ERR;
    }
    return ANET_OK;
}
```

bind用于将socket文件描述符绑定到地址，而listen则是准备接受连接并指定最大连接数量。在执行anetTcpServer之后会执行anetNonBlock函数，该函数调用fcntl将文件描述符设置为 O_NONBLOCK 。

在完成这一过程之后，server->ipfd_count中保存了监听的socket数量，在server->ipfd数组中保存了所有socket的文件描述符。



### 时间事件

目前redis的时间事件只有运行serverCron。

调用aeCreateTimeEvent将其注册到事件循环中。

``` c
    /* Create the timer callback, this is our way to process many background
     * operations incrementally, like clients timeout, eviction of unaccessed
     * expired keys and so forth. */
    if (aeCreateTimeEvent(server.el, 1, serverCron, NULL, NULL) == AE_ERR) {
        serverPanic("Can't create event loop timers.");
        exit(1);
    }
```

### 文件事件

对于server->ipfd中的所有socketfd，调用aeCreateFileEvent将其注册到事件循环。

``` c
    for (j = 0; j < server.ipfd_count; j++) {
        if (aeCreateFileEvent(server.el, server.ipfd[j], AE_READABLE,
            acceptTcpHandler,NULL) == AE_ERR)
            {
                serverPanic(
                    "Unrecoverable error creating server.ipfd file event.");
            }
    }
```

aeCreateFileEvent函数的实现在ae.c文件中。

监听的事件为AE_READABLE，在使用epoll时其对应的事件为EPOLLIN，且对应的系统调用为epoll_ctl。（epoll_ctl的参数为epfd,OP,fd和epoll_event）。

在注册到epoll成功后，再对**eventLoop->events[fd]**进行设置(events为初始化容量setsize为maxclients加上一个常数大小的数组，而fd是可复用的，所以redis可以同时支持的最大连接数为setsize减去其它保留的文件描述符数量)

eventLoop->events[fd]为FileEvent结构体，包含mask(事件类型)，rfileProc(读事件处理函数)，wfileProc(写事件处理函数) 和 clientData（此处为空NULL）。其中rfileProc和wfileProc都是 acceptTcpHandler。

在主事件循环中，wait到事件发生之后，会根据事件的fd找到对应的epollLoop->events[fd]，然后后根据mask，调用对应的处理函数rfileProc或者wfileProc。



### acceptTcpHandler

> ``` c
> void acceptTcpHandler(aeEventLoop *el, int fd, void *privdata, int mask) {
>     int cport, cfd, max = MAX_ACCEPTS_PER_CALL;
>     char cip[NET_IP_STR_LEN];
>     UNUSED(el);
>     UNUSED(mask);
>     UNUSED(privdata);
> 
>     while(max--) {
>         cfd = anetTcpAccept(server.neterr, fd, cip, sizeof(cip), &cport);
>         if (cfd == ANET_ERR) {
>             if (errno != EWOULDBLOCK)
>                 serverLog(LL_WARNING,
>                     "Accepting client connection: %s", server.neterr);
>             return;
>         }
>         serverLog(LL_VERBOSE,"Accepted %s:%d", cip, cport);
>         acceptCommonHandler(connCreateAcceptedSocket(cfd),0,cip);
>     }
> }
> ```



在监听的地址(server->ipfd[i])上有READABLE事件发生时，就会调用这个函数。每次调用最多处理MAX_ACCEPTS_PER_CALL个传入连接。

anetTcpAccept是对accept()系统调用的封装。由于在此前将已经监听的socket文件描述符fd设置为NONBLOCK了，在没有传入连接时accept不会阻塞而是会返回文件描述符-1 。

如果没有可用连接，会在while(max--)的循环中return，如果有可用连接，则调用acceptCommonHandler来处理：

> acceptCommonHandler(connCreateAcceptedSocket(cfd),0,cip);

 其中 connCreateAcceptedSocket是将文件描述符和状态封装到一个conn结构体中的函数。

 **connCreateAcceptedSocket**会调用connCreateSocket创建一个conn结构体，并将conn->type设置为全局变量的指针(单例？) connection.c/CT_Socket。

CT_SOCKET的定义如下(connections.c)

``` c
ConnectionType CT_Socket = {
    .ae_handler = connSocketEventHandler,
    .close = connSocketClose,
    .write = connSocketWrite,
    .read = connSocketRead,
    .accept = connSocketAccept,
    .connect = connSocketConnect,
    .set_write_handler = connSocketSetWriteHandler,
    .set_read_handler = connSocketSetReadHandler,
    .get_last_error = connSocketGetLastError,
    .blocking_connect = connSocketBlockingConnect,
    .sync_write = connSocketSyncWrite,
    .sync_read = connSocketSyncRead,
    .sync_readline = connSocketSyncReadLine
};
```



#### acceptCommonHandler

``` c
static void acceptCommonHandler(connection *conn, int flags, char *ip) {
    client *c;
    UNUSED(ip);
    if (listLength(server.clients) >= server.maxclients) {
        char *err = "-ERR max number of clients reached\r\n";
		//省略err handle
        return;
    }

    /* Create connection and client */
    if ((c = createClient(conn)) == NULL) {
		//省略err handle
        return;
    }
    c->flags |= flags;
    if (connAccept(conn, clientAcceptHandler) == C_ERR) {
		//省略err handle
        return;
    }
}
```

主要是两个函数：

- createClient

  完成conn的创建之后(设置fd等并将conn->type设置为&CT_Socket)，将其作为acceptCommonHandler的参数传入。在acceptCommonHandler函数中创建client对象并将client->conn设置为conn。之后将client添加到server中的clients链表和索引client_index(radix树，用于加快删除速度)。

  > createClient 片段
  >
  > ``` c
  >     if (conn) {
  >         connNonBlock(conn);
  >         connEnableTcpNoDelay(conn);
  >         if (server.tcpkeepalive)
  >             connKeepAlive(conn,server.tcpkeepalive);
  >         connSetReadHandler(conn, readQueryFromClient);
  >         connSetPrivateData(conn, c);
  >     }
  > ```
  >
  > 其中的**connSetReadHandler**会调用conn->type->set_read_handler，最终调用aeCreateFileEvent将 **readQueryFromClient**作为该fd的读事件处理函数注册到eventLoop中

- connAccept

  立即调用conn->type->accept 。由于此处的type为CT_Socket，会调用

  > ``` c
  > static int connSocketAccept(connection *conn, ConnectionCallbackFunc accept_handler) {
  >     if (conn->state != CONN_STATE_ACCEPTING) return C_ERR;
  >     conn->state = CONN_STATE_CONNECTED;
  >     if (!callHandler(conn, accept_handler)) return C_ERR;
  >     return C_OK;
  > }
  > ```

  在callHandler函数中最终调用的还是accept_handler。可见绕了一圈又回来了。在增加了一些错误处理的逻辑，包括在检测到连接断开时调用type->close，close中调用aeDeleteFileEvent将对应fd的读/写事件从事件循环中删除并在close(fd)系统调用之后将conn->fd重置为-1等。而clientAcceptHandler中也只是检测连接是否存活的逻辑。



#### readQueryFromClient

如果postponeClientRead(c)的结果为1表明使用采用多线程处理，则不会执行后续步骤而是将需要处理的客户端添加到server.clients_pending_read链表中。在处理完事件之后使用多线程IO进行读取。

将最多min(sdsfree,PROTO_IOBUF_LEN)个字节的数据读取到client->querybuf的末尾，然后将sds扩容读取到的字节数，如果超过最大限制，断开该客户端的连接（实际上是调用freeClientAsync将关闭事件放到serverCron函数的调度中去执行）。

最后调用processInputBufferAndReplicate。这个函数会调用processInlineBuffer，先将querybuf中的数据解析，然后设置client->cmd，client->argv和client->argc的值，然后调用processCommandAndResetClient执行相应的命令。



#### 回复客户端

在将客户端回复写入到client->buf中之后并不会立即回复客户端。而在beforeSleep中有这样一个函数调用：

``` c
    /* Handle writes with pending output buffers. */
    handleClientsWithPendingWritesUsingThreads();
	// redis6.0中加入的多线程IO，单线程的版本为handleClientsWithPendingWrites
```

在该函数中，会调用connSetWriteHandler，将sendReplyToClient函数注册到事件循环中。



### aeMain

``` c
void aeMain(aeEventLoop *eventLoop) {
    eventLoop->stop = 0;
    while (!eventLoop->stop) {
        if (eventLoop->beforesleep != NULL)
            eventLoop->beforesleep(eventLoop);
        aeProcessEvents(eventLoop, AE_ALL_EVENTS|AE_CALL_AFTER_SLEEP);
    }
}
```

aeMain事件主循环中，在调用aeProcessEvents之前会调用beforeSleep，所以在一个事件循环中处理过的命令，其回复事件会在下一次事件循环之前注册到eventLoop，再从aeProcessEvents中监听到相应事件执行。





### redis处理客户端响应的大致流程

- redis启动，打开socket，将fd绑定到指定地址。
- 初始化事件循环，将该fd的传入事件和相应处理函数acceptTcpHandler注册到事件循环。
- 开启事件循环。
- 连接传入，调用acceptTcpHandler进行处理
  - 为该连接初始化一个client结构体，将该连接的fd和对应读事件处理函数readQueryFromClient注册到事件循环。
- 收到来自客户端的数据，触发AE_READABLE事件，调用readQueryFromClient进行处理。
  - readQueryFromClient将数据读取到client.querybuf中。如果本次读取完了所有数据，则将其解析，填充clent.cmd，client.argc和client.argv并设置客户端状态。
  - 根据client.cmd执行该命令。将结果保存到client.buf中
- 处理读取的事件循环结束，在下一个事件循环开始之前会调用beforeSleep函数。（beforeSleep在server.c中定义，然后在main函数中使用asSetBeforeSleepProc将其设置到eventLoop中）
- beforeSleep中再将对应写事件sendReplyToClient注册到eventLoop中，这样在接下来的循环中就会触发事件进行写入。





