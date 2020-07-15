# 网络通信

## 简介

---
源码位置：anet.c/anet.h/networking.c

**1. 简介**
Redis在`anet.c`中对TCP/IP网络中socket api接口和状态设置进行了封装。状态设置主要包括socket连接的阻塞性、tcp的保活定时器的设置、设置发送缓冲区、tcp的nagle算法设置、设置发送/接收超时时间、地址重用的设置和IPv6/IPv4的设置等。
Redis网络通讯的具体实现在`networking.c`中，主要包括如何建立和客户端的连接，并且接收其命令，返回给客户端。

**2. 回顾tcp socket编程**
2.1 TCP客户/服务器程序socket编程流程如下：
![socket](../img/stage4/socket.png)

2.2 TCP的三次握手
![handshake](../img/stage4/tcp_handshake.png)

2.3 TCP的四次挥手
TCP的断开连接操作可由**任意一端发起**
![close](../img/stage4/tcp_close.png)

**3. anet解析**
anet.h中定义的函数如下：

```c
int anetTcpConnect(char *err, const char *addr, int port); // tcp连接
int anetTcpNonBlockConnect(char *err, const char *addr, int port); // TCP非阻塞连接
int anetTcpNonBlockBindConnect(char *err, const char *addr, int port, const char *source_addr); // TCP非阻塞绑定
int anetTcpNonBlockBestEffortBindConnect(char *err, const char *addr, int port, const char *source_addr); // TCP非阻塞绑定
int anetUnixConnect(char *err, const char *path); // unix socket连接
int anetUnixNonBlockConnect(char *err, const char *path); // unix socket非阻塞连接
int anetRead(int fd, char *buf, int count); // socket读数据
int anetResolve(char *err, char *host, char *ipbuf, size_t ipbuf_len); // 解析所有的东西
int anetResolveIP(char *err, char *host, char *ipbuf, size_t ipbuf_len); // 解析IP
int anetTcpServer(char *err, int port, char *bindaddr, int backlog); // IPv4下创建socket
int anetTcp6Server(char *err, int port, char *bindaddr, int backlog); // IPv6下创建socket
int anetUnixServer(char *err, char *path, mode_t perm, int backlog); // unix创建socket和bind
int anetTcpAccept(char *err, int serversock, char *ip, size_t ip_len, int *port); // tcp socket接收
int anetUnixAccept(char *err, int serversock); // unix tcp socket接收
int anetWrite(int fd, char *buf, int count); // socket写数据
int anetNonBlock(char *err, int fd); // 设置socket为非阻塞
int anetBlock(char *err, int fd); // 设置socket为阻塞
int anetEnableTcpNoDelay(char *err, int fd); // 启用tcp_nodelay选项（关闭Nagle算法）
int anetDisableTcpNoDelay(char *err, int fd); // 禁用tcp_nodelay选项
int anetTcpKeepAlive(char *err, int fd); // 设置tcp keepalive选项
int anetSendTimeout(char *err, int fd, long long ms); // 设置发送超时时间
int anetRecvTimeout(char *err, int fd, long long ms); // 设置接收超时时间
int anetPeerToString(int fd, char *ip, size_t ip_len, int *port); // 获取客户端的ip、port
int anetKeepAlive(char *err, int fd, int interval); // 设置tcp keepalive选项
int anetSockName(int fd, char *ip, size_t ip_len, int *port); // 获取套接字的名字
/* 格式化操作 */
int anetFormatAddr(char *fmt, size_t fmt_len, char *ip, int port);
int anetFormatPeer(int fd, char *fmt, size_t fmt_len);
int anetFormatSock(int fd, char *fmt, size_t fmt_len);
```

Redis服务启动后会调用`server.c/listenToPort()`进行socket相关的设置和端口监听。如果服务器配置不包含要绑定的特定地址，则该函数会尝试IPv6（调用`anetTcp6Server()`）和IPv4（调用`anetTcpServer()`）协议进行绑定。
不管是使用IPv6还是IPv4协议，最终调用的都是`_anetTcpServer()`来创建socket并进行绑定监听，以下是该函数的实现代码：

```c
static int _anetTcpServer(char *err, int port, char *bindaddr, int af, int backlog)
{
    int s = -1, rv;
    char _port[6];  /* strlen("65535") */
    struct addrinfo hints, *servinfo, *p;

    snprintf(_port,6,"%d",port);
    memset(&hints,0,sizeof(hints));
    hints.ai_family = af;
    hints.ai_socktype = SOCK_STREAM;
    hints.ai_flags = AI_PASSIVE;    /* No effect if bindaddr != NULL */

    // 将域名解析成ip地址
    if ((rv = getaddrinfo(bindaddr,_port,&hints,&servinfo)) != 0) {
        anetSetError(err, "%s", gai_strerror(rv));
        return ANET_ERR;
    }
    for (p = servinfo; p != NULL; p = p->ai_next) {
        if ((s = socket(p->ai_family,p->ai_socktype,p->ai_protocol)) == -1)
            continue;

        if (af == AF_INET6 && anetV6Only(err,s) == ANET_ERR) goto error;
        // 设置SO_REUSEADDR允许我们重复bind相同的本地地址
        if (anetSetReuseAddr(err,s) == ANET_ERR) goto error;
        // bind && listen
        if (anetListen(err,s,p->ai_addr,p->ai_addrlen,backlog) == ANET_ERR) s = ANET_ERR;
        goto end;
    }
    if (p == NULL) {
        anetSetError(err, "unable to bind socket, errno: %d", errno);
        goto error;
    }

error:
    if (s != -1) close(s);
    s = ANET_ERR;
end:
    freeaddrinfo(servinfo);
    return s;
}
```

**4. networking解析**
4.1 建立连接
Redis服务端会初始化一个socket端口来监听客户端的连接，当一个连接建立后，服务端会对客户端的socket进行设置：

1. 客户端socket设置为非阻塞模式，因为Redis采用的是非阻塞I/O多路复用模型。
2. 客户端socket设置为 TCP_NODELAY 属性，禁用 Nagle 算法。
3. 将该socket绑定读事件到时间loop，用于监听这个客户端socket的数据发送。
4. 建立连接后如果发现已经超过最大连接数，则关闭连接，删除该客户端socket。

```c
client *createClient(connection *conn) {
    client *c = zmalloc(sizeof(client));

    /*
     * 当conn为NULL时，创建无网络连接的伪客户端
     * 当conn不为NULL时，创建带网络连接的客户端
     * 因为 Redis 的命令必须在客户端的上下文中使用，所以在执行 Lua 环境中的命令时
     * 需要用到这种伪终端
     */
    if (conn) {
        // 设置非阻塞
        connNonBlock(conn);
        // 禁用tcp_nodelay
        connEnableTcpNoDelay(conn);
        // 设置 keep alive
        if (server.tcpkeepalive)
            connKeepAlive(conn,server.tcpkeepalive);
        // 绑定读事件到事件loop（开始接收命令请求）
        connSetReadHandler(conn, readQueryFromClient);
        // 将私有数据指针与连接相关联
        connSetPrivateData(conn, c);
    }

    // 初始化客户端参数
    selectDb(c,0);
    uint64_t client_id = ++server.next_client_id;
    c->id = client_id;
    c->resp = 2;
    c->conn = conn;
    c->name = NULL;
    c->bufpos = 0;
    c->qb_pos = 0;
    c->querybuf = sdsempty();
    c->pending_querybuf = sdsempty();
    c->querybuf_peak = 0;
    c->reqtype = 0;
    c->argc = 0;
    c->argv = NULL;
    c->cmd = c->lastcmd = NULL;
    c->user = DefaultUser;
    c->multibulklen = 0;
    c->bulklen = -1;
    c->sentlen = 0;
    c->flags = 0;
    c->ctime = c->lastinteraction = server.unixtime;
    /* If the default user does not require authentication, the user is
     * directly authenticated. */
    c->authenticated = (c->user->flags & USER_FLAG_NOPASS) != 0;
    c->replstate = REPL_STATE_NONE;
    c->repl_put_online_on_ack = 0;
    c->reploff = 0;
    c->read_reploff = 0;
    c->repl_ack_off = 0;
    c->repl_ack_time = 0;
    c->slave_listening_port = 0;
    c->slave_ip[0] = '\0';
    c->slave_capa = SLAVE_CAPA_NONE;
    c->reply = listCreate();
    c->reply_bytes = 0;
    c->obuf_soft_limit_reached_time = 0;
    listSetFreeMethod(c->reply,freeClientReplyValue);
    listSetDupMethod(c->reply,dupClientReplyValue);
    c->btype = BLOCKED_NONE;
    c->bpop.timeout = 0;
    c->bpop.keys = dictCreate(&objectKeyHeapPointerValueDictType,NULL);
    c->bpop.target = NULL;
    c->bpop.xread_group = NULL;
    c->bpop.xread_consumer = NULL;
    c->bpop.xread_group_noack = 0;
    c->bpop.numreplicas = 0;
    c->bpop.reploffset = 0;
    c->woff = 0;
    c->watched_keys = listCreate();
    c->pubsub_channels = dictCreate(&objectKeyPointerValueDictType,NULL);
    c->pubsub_patterns = listCreate();
    c->peerid = NULL;
    c->client_list_node = NULL;
    c->client_tracking_redirection = 0;
    c->client_tracking_prefixes = NULL;
    c->auth_callback = NULL;
    c->auth_callback_privdata = NULL;
    c->auth_module = NULL;
    listSetFreeMethod(c->pubsub_patterns,decrRefCountVoid);
    listSetMatchMethod(c->pubsub_patterns,listMatchObjects);
    if (conn) linkClient(c); // 如果不是伪客户端，那么添加到服务器的客户端链表中
    initClientMultiState(c); // 初始化客户端的事务状态
    return c;
}
```

4.2 接收处理请求
