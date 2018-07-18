---
layout: blog
article: true
category: C++
title:  通过UNIX域套接字传递文件描述符
date:   2018-07-17 23:36:47
tags:
- Nebula系列
---


&emsp;&emsp;传送文件描述符是高并发网络服务编程的一种常见实现方式。[Nebula](https://github.com/Bwar/Nebula) 高性能通用网络框架即采用了UNIX域套接字传递文件描述符设计和实现。本文详细说明一下传送文件描述符的应用。

### 1. TCP服务器程序设计范式
&emsp;&emsp;开发一个服务器程序，有较多的的程序设计范式可供选择，不同范式有其自身的特点和实用范围，明了不同范式的特性有助于我们服务器程序的开发。常见的TCP服务器程序设计范式有以下几种：
* __迭代服务器__
* __并发服务器，每个客户请求fork一个子进程__ 
* __预先派生子进程，每个子进程无保护地调用accept__ 
* __预先派生子进程,使用文件上锁保护accept__ 
* __预先派生子进程,使用线程互斥锁上锁保护accept__
* __预先派生子进程，父进程向子进程传递套接字描述符__
* __并发服务器，每个客户端请求创建一个线程__
* __预先创建线程服务器，使用互斥锁上锁保护accept__
* __预先创建线程服务器，由主线程调用accept__

&emsp;&emsp;当系统负载较轻时，传统的并发服务器程序模型就够了。相对于传统的每个客户一次fork设计，预先创建一个进程池或线程池可以减少进程控制CPU时间，大约可减少10倍以上。

&emsp;&emsp;某些实现允许多个子进程或线程阻塞在accept上，然而在另一些实现中，我们必须使用文件锁、线程互斥锁或其他类型的锁来确保每次只有一个子进程或线程在accept。

&emsp;&emsp;一般来讲，所有子进程或线程都调用accept要比父进程或主线程调用accept后将描述字传递个子进程或线程来得快且简单。

### 2. [Nebula](https://github.com/Bwar/Nebula) 为什么采用传递文件描述符方式？
&emsp;&emsp;Nebula框架是预先创建多进程，由Manager主进程accept后传递文件描述符到Worker子进程的服务模型（[Nebula进程模型](https://github.com/Bwar/Nebula/wiki/%E5%B7%A5%E4%BD%9C%E5%8E%9F%E7%90%86)）。为什么不采用像nginx那样多线程由子线程使用互斥锁上锁保护accept的服务模型？而且这种服务模型的实现比传递文件描述符来得还简单一些。

&emsp;&emsp;Nebula框架采用无锁设计，进程之前完全不共享数据，不存在需要互斥访问的地方。没错，会存在数据多副本问题，但这些多副本往往只是些配置数据，占用不了太大内存，与加锁解锁带来的代码复杂度及锁开销相比这点内存代价更划算也更简单。

&emsp;&emsp;同一个Nebula服务的工作进程间不相互通信，采用进程和线程并无太大差异，之所以采用进程而不是线程的最重要考虑是Nebula是出于稳定性和容错性考虑。Nebula是通用框架，完全业务无关，业务都是通过动态加载的方式或通过将Nebula链接进业务Server的方式来实现。Nebula框架无法预知业务代码的质量，但可以保证在服务因业务代码导致coredump或其他情况时，框架可以实时监控到并立刻拉起服务进程，最大程度保障服务可用性。

&emsp;&emsp;决定Nebula采用传递文件描述符方式的最重要一点是：Nebula定位是高性能分布式服务集群解决方案的基础通信框架，其设计更多要为构建分布式服务集群而考虑。集群不同服务节点之间通过TCP通信，而所有逻辑都是Worker进程负责，这意味着节点之间通信需要指定到Worker进程，而如果采用子进程竞争accept的方式无法保证指定的子进程获得资源，那么第一个通信数据包将会路由错误。采用传递文件描述符方式可以很完美地解决这个问题，而且传递文件描述符也非常高效。

### 3. 文件描述符传递函数和数据结构
&emsp;&emsp;文件描述符传递通过调用sendmsg()函数发送，调用recvmsg()函数接收：

``` c
#include <sys/types.h>
#include <sys/socket.h>

ssize_t sendmsg(int sockfd, const struct msghdr *msg, int flags);
ssize_t recvmsg(int sockfd, struct msghdr *msg, int flags);
```

&emsp;&emsp;这两个函数与sendto和recvfrom函数相似，只不过可以传输更复杂的数据结构，不仅可以传输一般数据，还可以传输额外的数据，即文件描述符。下面来看结构体msghdr及其相关结构体 ：

``` c
struct msghdr {
    void         *msg_name;       /* optional address */
    socklen_t     msg_namelen;    /* size of address */
    struct iovec *msg_iov;        /* scatter/gather array */
    size_t        msg_iovlen;     /* # elements in msg_iov */
    void         *msg_control;    /* ancillary data, see below */
    size_t        msg_controllen; /* ancillary data buffer len */
    int           msg_flags;      /* flags on received message */
};


/* iovec结构体 */
struct iovec {
    void  *iov_base;    /* Starting address */
    size_t iov_len;     /* Number of bytes to transfer */
};


/* cmsghdr结构体 */
struct cmsghdr {
    socklen_t cmsg_len;    /* data byte count, including header */
    int       cmsg_level;  /* originating protocol */
    int       cmsg_type;   /* protocol-specific type */
    /* followed by unsigned char cmsg_data[]; */
};
```

&emsp;&emsp;msghdr结构成员说明：
* msg_name ：即对等方的地址指针，不关心时设为NULL即可.
* msg_namelen：地址长度，不关心时设置为0即可.
* msg_iov：结构体iovec指针;   iovec的成员iov_base 可以认为是传输正常数据时的buf，iov_len 是buf 的大小。
* msg_iovlen：iovec类型的元素的个数，每一个缓冲区的起始地址和大小由iovec类型自包含。当有n个iovec结构体时，此值为n。
* msg_control：是一个指向cmsghdr 结构体的指针，用来发送或接收控制信息。
* msg_controllen ：控制信息所占用的字节数。注意，msg_controllen与前面的msg_iovlen不同，msg_iovlen是指的由成员msg_iov所指向的iovec型的数组的元素个数，而msg_controllen，则是所有控制信息所占用的总的字节数。
* msg_flags ： 用来描述接受到的消息的性质，由调用recvmsg时传入的flags参数设置。

&emsp;&emsp;为了对齐，可能存在一些填充字节，跟不同系统的实现有关控制信息的数据部分，是直接存储在cmsghdr结构体的cmsg_type之后的。但中间可能有一些由于对齐产生的填充字节，由于这些填充数据的存在，对于这些控制数据的访问，必须使用Linux提供的一些专用宏来完成：

``` c
#include <sys/socket.h>

/* 返回msgh所指向的msghdr类型的缓冲区中的第一个cmsghdr结构体的指针。*/
struct cmsghdr *CMSG_FIRSTHDR(struct msghdr *msgh);

/* 返回传入的cmsghdr类型的指针的下一个cmsghdr结构体的指针。 */
struct cmsghdr *CMSG_NXTHDR(struct msghdr *msgh, struct cmsghdr *cmsg);

/* 根据传入的length大小,返回一个包含了添加对齐作用的填充数据后的大小。 */
size_t CMSG_ALIGN(size_t length);

/* 传入的参数length指的是一个控制信息元素(即一个cmsghdr结构体)后面数据部分的字节数,返回的是这个控制信息的总的字节数,即包含了头部(即cmsghdr各成员)、数据部分和填充数据的总和。*/
size_t CMSG_SPACE(size_t length);

/* 根据传入的cmsghdr指针参数，返回其后面数据部分的指针。*/
size_t CMSG_LEN(size_t length);

/* 传入的参数是一个控制信息中的数据部分的大小，返回的是这个根据这个数据部分大小，需要配置的cmsghdr结构体中cmsg_len成员的值。这个大小将为对齐添加的填充数据也包含在内。*/
unsigned char *CMSG_DATA(struct cmsghdr *cmsg);
```

### 4. 文件描述符传递要点
&emsp;&emsp;sendmsg提供了可以传递控制信息的功能，要实现的传递描述符这一功能必须要用到这个控制信息。在msghdr变量的cmsghdr成员中，由控制头cmsg_level和cmsg_type来设置传递文件描述符这一属性，并将要传递的文件描述符作为数据部分，保存在cmsghdr变量的后面。这样就可以实现传递文件描述符这一功能，这种情况是不需要使用msg_iov来传递数据的。

&emsp;&emsp;具体地说，为msghdr的成员msg_control分配一个cmsghdr的空间，将该cmsghdr结构的cmsg_level设置为SOL_SOCKET，cmsg_type设置为SCM_RIGHTS，并将要传递的文件描述符作为数据部分，调用sendmsg即可。其中SCM表示socket-level control message，SCM_RIGHTS表示我们要传递访问权限。

&emsp;&emsp;跟发送部分一样，为控制信息配置好属性，并在其后分配一个文件描述符的数据部分后，在成功调用recvmsg后，控制信息的数据部分就是在接收进程中的新的文件描述符了，接收进程可直接对该文件描述符进行操作。

&emsp;&emsp;文件描述符传递并不是将文件描述符数字传递，而是文件描述符对应数据结构。在主进程accept的到的文件描述符7传递到子进程后文件描述符有可能是7，更有可能是7以外的其他数值，但无论是什么数值并不重要，重要的是传递之后的连接跟传递之前的连接是同一个连接。

&emsp;&emsp;通常在完成文件描述符传递后，接收进程接管文件描述符，发送进程则应调用close关闭已传递的文件描述符。发送进程关闭描述符并不造成关闭该文件或设备，因为该描述符对应的文件仍被视为由接收者进程打开（即使接收进程尚未接收到该描述符）。

&emsp;&emsp;文件描述符传递可经由基于STREAMS的管道，也可经由UNIX域套接字。两种方式在《UNIX网络编程》中均有描述，Nebula采用的UNIX域套接字传递文件描述符。

&emsp;&emsp;创建用于传递文件描述符的UNIX域套接字用到socketpair函数：

``` c
#include <sys/types.h>  
#include <sys/socket.h>  
  
int socketpair(int d, int type, int protocol, int sv[2]);  
```

&emsp;&emsp;传入的参数sv为一个整型数组，有两个元素。当调用成功后，这个数组的两个元素即为2个文件描述符。一对连接起来的Unix匿名域套接字就建立起来了，它们就像一个全双工的管道，每一端都既可读也可写。

### 5. Nebula框架中传递文件描述符的实现
&emsp;&emsp;Nebula框架的文件描述符属于[SocketChannel](https://github.com/Bwar/Nebula/blob/master/src/channel/SocketChannel.hpp)的基本属性，文件描述符传递方法是SocketChannel的静态方法。

文件描述符传递方法声明：

``` C++
static int SendChannelFd(int iSocketFd, int iSendFd, int iCodecType, std::shared_ptr<NetLogger> pLogger);
static int RecvChannelFd(int iSocketFd, int& iRecvFd, int& iCodecType, std::shared_ptr<NetLogger> pLogger);
```

文件描述符发送方法实现：

``` C++
/**
* @brief 发送文件描述符
* @param iSocketFd 由socketpair()创建的UNIX域套接字，用于传递文件描述符
* @param iSendFd 待发送的文件描述符
* @param iCodecType 通信通道编解码类型
* @param pLogger 日志类指针
* @return errno 错误码
*/
int SocketChannel::SendChannelFd(int iSocketFd, int iSendFd, int iCodecType, std::shared_ptr<NetLogger> pLogger)
{
    ssize_t             n;
    struct iovec        iov[1];
    struct msghdr       msg;
    tagChannelCtx stCh;
    int iError = 0;

    stCh.iFd = iSendFd;
    stCh.iCodecType = iCodecType;

    union
    {
        struct cmsghdr  cm;
        char            space[CMSG_SPACE(sizeof(int))];
    } cmsg;

    if (stCh.iFd == -1)
    {
        msg.msg_control = NULL;
        msg.msg_controllen = 0;

    }
    else
    {
        msg.msg_control = (caddr_t) &cmsg;
        msg.msg_controllen = sizeof(cmsg);

        memset(&cmsg, 0, sizeof(cmsg));

        cmsg.cm.cmsg_len = CMSG_LEN(sizeof(int));
        cmsg.cm.cmsg_level = SOL_SOCKET;
        cmsg.cm.cmsg_type = SCM_RIGHTS;

        *(int *) CMSG_DATA(&cmsg.cm) = stCh.iFd;
    }

    msg.msg_flags = 0;

    iov[0].iov_base = (char*)&stCh;
    iov[0].iov_len = sizeof(tagChannelCtx);

    msg.msg_name = NULL;
    msg.msg_namelen = 0;
    msg.msg_iov = iov;
    msg.msg_iovlen = 1;

    n = sendmsg(iSocketFd, &msg, 0);

    if (n == -1)
    {
        pLogger->WriteLog(neb::Logger::ERROR, __FILE__, __LINE__, __FUNCTION__, "sendmsg() failed, errno %d", errno);
        iError = (errno == 0) ? ERR_TRANSFER_FD : errno;
        return(iError);
    }

    return(ERR_OK);
}
```

文件描述符接收方法实现：

``` C++
/**
* @brief 接收文件描述符
* @param iSocketFd 由socketpair()创建的UNIX域套接字，用于传递文件描述符
* @param iRecvFd 接收到的文件描述符
* @param iCodecType 接收到的通信通道编解码类型
* @param pLogger 日志类指针
* @return errno 错误码
*/
int SocketChannel::RecvChannelFd(int iSocketFd, int& iRecvFd, int& iCodecType, std::shared_ptr<NetLogger> pLogger)
{
    ssize_t             n;
    struct iovec        iov[1];
    struct msghdr       msg;
    tagChannelCtx stCh;
    int iError = 0;

    union {
        struct cmsghdr  cm;
        char            space[CMSG_SPACE(sizeof(int))];
    } cmsg;

    iov[0].iov_base = (char*)&stCh;
    iov[0].iov_len = sizeof(tagChannelCtx);

    msg.msg_name = NULL;
    msg.msg_namelen = 0;
    msg.msg_iov = iov;
    msg.msg_iovlen = 1;

    msg.msg_control = (caddr_t) &cmsg;
    msg.msg_controllen = sizeof(cmsg);

    n = recvmsg(iSocketFd, &msg, 0);

    if (n == -1) {
        pLogger->WriteLog(neb::Logger::ERROR, __FILE__, __LINE__, __FUNCTION__, "recvmsg() failed, errno %d", errno);
        iError = (errno == 0) ? ERR_TRANSFER_FD : errno;
        return(iError);
    }

    if (n == 0) {
        pLogger->WriteLog(neb::Logger::ERROR, __FILE__, __LINE__, __FUNCTION__, "recvmsg() return zero, errno %d", errno);
        iError = (errno == 0) ? ERR_TRANSFER_FD : errno;
        return(ERR_CHANNEL_EOF);
    }

    if ((size_t) n < sizeof(tagChannelCtx))
    {
        pLogger->WriteLog(neb::Logger::ERROR, __FILE__, __LINE__, __FUNCTION__, "rrecvmsg() returned not enough data: %z, errno %d", n, errno);
        iError = (errno == 0) ? ERR_TRANSFER_FD : errno;
        return(iError);
    }

    if (cmsg.cm.cmsg_len < (socklen_t) CMSG_LEN(sizeof(int)))
    {
        pLogger->WriteLog(neb::Logger::ERROR, __FILE__, __LINE__, __FUNCTION__, "recvmsg() returned too small ancillary data");
        iError = (errno == 0) ? ERR_TRANSFER_FD : errno;
        return(iError);
    }

    if (cmsg.cm.cmsg_level != SOL_SOCKET || cmsg.cm.cmsg_type != SCM_RIGHTS)
    {
        pLogger->WriteLog(neb::Logger::ERROR, __FILE__, __LINE__, __FUNCTION__,
                        "recvmsg() returned invalid ancillary data level %d or type %d", cmsg.cm.cmsg_level, cmsg.cm.cmsg_type);
        iError = (errno == 0) ? ERR_TRANSFER_FD : errno;
        return(iError);
    }

    stCh.iFd = *(int *) CMSG_DATA(&cmsg.cm);

    if (msg.msg_flags & (MSG_TRUNC|MSG_CTRUNC))
    {
        pLogger->WriteLog(neb::Logger::ERROR, __FILE__, __LINE__, __FUNCTION__, "recvmsg() truncated data");
        iError = (errno == 0) ? ERR_TRANSFER_FD : errno;
        return(iError);
    }

    iRecvFd = stCh.iFd;
    iCodecType = stCh.iCodecType;

    return(ERR_OK);
}
```

[Manager](https://github.com/Bwar/Nebula/blob/master/src/labor/Manager.cpp)进程的void Manager::CreateWorker()方法创建用于传递文件描述符的UNIX域套接字：

``` C++
int iControlFds[2];
int iDataFds[2];
if (socketpair(PF_UNIX, SOCK_STREAM, 0, iControlFds) < 0)
{
    LOG4_ERROR("error %d: %s", errno, strerror_r(errno, m_szErrBuff, 1024));
}
if (socketpair(PF_UNIX, SOCK_STREAM, 0, iDataFds) < 0)
{
    LOG4_ERROR("error %d: %s", errno, strerror_r(errno, m_szErrBuff, 1024));
}
```

Manager进程发送文件描述符：

``` C++
int iCodec = m_stManagerInfo.eCodec;    // 将编解码方式和文件描述符一同发送给Worker进程
int iErrno = SocketChannel::SendChannelFd(worker_pid_fd.second, iAcceptFd, iCodec, m_pLogger);
if (iErrno == 0)
{
    AddWorkerLoad(worker_pid_fd.first);
}
else
{
    LOG4_ERROR("error %d: %s", iErrno, strerror_r(iErrno, m_szErrBuff, 1024));
}
close(iAcceptFd);   // 发送完毕，关闭文件描述符
```

Worker进程接收文件描述符：

``` C++
int iAcceptFd = -1;
int iCodec = 0;     // 这里的编解码方式在RecvChannelFd方法中获得
int iErrno = SocketChannel::RecvChannelFd(m_stWorkerInfo.iManagerDataFd, iAcceptFd, iCodec, m_pLogger);
```

&emsp;&emsp;至此，Nebula框架的文件描述符传递分享完毕，下面再看看nginx中的文件描述符传递实现。

### 6. Nginx文件描述符传递代码实现
&emsp;&emsp;Nginx的文件描述符传递代码在os/unix/ngx_channel.c文件中。

nginx中发送文件描述符代码：

``` c
ngx_int_t
ngx_write_channel(ngx_socket_t s, ngx_channel_t *ch, size_t size,
    ngx_log_t *log)
{
    ssize_t             n;
    ngx_err_t           err;
    struct iovec        iov[1];
    struct msghdr       msg;

#if (NGX_HAVE_MSGHDR_MSG_CONTROL)

    union {
        struct cmsghdr  cm;
        char            space[CMSG_SPACE(sizeof(int))];
    } cmsg;

    if (ch->fd == -1) {
        msg.msg_control = NULL;
        msg.msg_controllen = 0;

    } else {
        msg.msg_control = (caddr_t) &cmsg;
        msg.msg_controllen = sizeof(cmsg);

        ngx_memzero(&cmsg, sizeof(cmsg));

        cmsg.cm.cmsg_len = CMSG_LEN(sizeof(int));
        cmsg.cm.cmsg_level = SOL_SOCKET;
        cmsg.cm.cmsg_type = SCM_RIGHTS;

        /*
         * We have to use ngx_memcpy() instead of simple
         *   *(int *) CMSG_DATA(&cmsg.cm) = ch->fd;
         * because some gcc 4.4 with -O2/3/s optimization issues the warning:
         *   dereferencing type-punned pointer will break strict-aliasing rules
         *
         * Fortunately, gcc with -O1 compiles this ngx_memcpy()
         * in the same simple assignment as in the code above
         */

        ngx_memcpy(CMSG_DATA(&cmsg.cm), &ch->fd, sizeof(int));
    }

    msg.msg_flags = 0;

#else

    if (ch->fd == -1) {
        msg.msg_accrights = NULL;
        msg.msg_accrightslen = 0;

    } else {
        msg.msg_accrights = (caddr_t) &ch->fd;
        msg.msg_accrightslen = sizeof(int);
    }

#endif

    iov[0].iov_base = (char *) ch;
    iov[0].iov_len = size;

    msg.msg_name = NULL;
    msg.msg_namelen = 0;
    msg.msg_iov = iov;
    msg.msg_iovlen = 1;

    n = sendmsg(s, &msg, 0);

    if (n == -1) {
        err = ngx_errno;
        if (err == NGX_EAGAIN) {
            return NGX_AGAIN;
        }

        ngx_log_error(NGX_LOG_ALERT, log, err, "sendmsg() failed");
        return NGX_ERROR;
    }

    return NGX_OK;
}
```

nginx中接收文件描述符代码：

``` c
ngx_int_t
ngx_read_channel(ngx_socket_t s, ngx_channel_t *ch, size_t size, ngx_log_t *log)
{
    ssize_t             n;
    ngx_err_t           err;
    struct iovec        iov[1];
    struct msghdr       msg;

#if (NGX_HAVE_MSGHDR_MSG_CONTROL)
    union {
        struct cmsghdr  cm;
        char            space[CMSG_SPACE(sizeof(int))];
    } cmsg;
#else
    int                 fd;
#endif

    iov[0].iov_base = (char *) ch;
    iov[0].iov_len = size;

    msg.msg_name = NULL;
    msg.msg_namelen = 0;
    msg.msg_iov = iov;
    msg.msg_iovlen = 1;

#if (NGX_HAVE_MSGHDR_MSG_CONTROL)
    msg.msg_control = (caddr_t) &cmsg;
    msg.msg_controllen = sizeof(cmsg);
#else
    msg.msg_accrights = (caddr_t) &fd;
    msg.msg_accrightslen = sizeof(int);
#endif

    n = recvmsg(s, &msg, 0);

    if (n == -1) {
        err = ngx_errno;
        if (err == NGX_EAGAIN) {
            return NGX_AGAIN;
        }

        ngx_log_error(NGX_LOG_ALERT, log, err, "recvmsg() failed");
        return NGX_ERROR;
    }

    if (n == 0) {
        ngx_log_debug0(NGX_LOG_DEBUG_CORE, log, 0, "recvmsg() returned zero");
        return NGX_ERROR;
    }

    if ((size_t) n < sizeof(ngx_channel_t)) {
        ngx_log_error(NGX_LOG_ALERT, log, 0,
                      "recvmsg() returned not enough data: %z", n);
        return NGX_ERROR;
    }

#if (NGX_HAVE_MSGHDR_MSG_CONTROL)

    if (ch->command == NGX_CMD_OPEN_CHANNEL) {

        if (cmsg.cm.cmsg_len < (socklen_t) CMSG_LEN(sizeof(int))) {
            ngx_log_error(NGX_LOG_ALERT, log, 0,
                          "recvmsg() returned too small ancillary data");
            return NGX_ERROR;
        }

        if (cmsg.cm.cmsg_level != SOL_SOCKET || cmsg.cm.cmsg_type != SCM_RIGHTS)
        {
            ngx_log_error(NGX_LOG_ALERT, log, 0,
                          "recvmsg() returned invalid ancillary data "
                          "level %d or type %d",
                          cmsg.cm.cmsg_level, cmsg.cm.cmsg_type);
            return NGX_ERROR;
        }

        /* ch->fd = *(int *) CMSG_DATA(&cmsg.cm); */

        ngx_memcpy(&ch->fd, CMSG_DATA(&cmsg.cm), sizeof(int));
    }

    if (msg.msg_flags & (MSG_TRUNC|MSG_CTRUNC)) {
        ngx_log_error(NGX_LOG_ALERT, log, 0,
                      "recvmsg() truncated data");
    }

#else

    if (ch->command == NGX_CMD_OPEN_CHANNEL) {
        if (msg.msg_accrightslen != sizeof(int)) {
            ngx_log_error(NGX_LOG_ALERT, log, 0,
                          "recvmsg() returned no ancillary data");
            return NGX_ERROR;
        }

        ch->fd = fd;
    }

#endif

    return n;
}
```

&emsp;&emsp;__Nebula框架系列技术分享__ 之 《通过UNIX域套接字传递文件描述符》。 如果觉得这篇文章对你有用，如果觉得Nebula框架还可以，帮忙到Nebula的[__Github__](https://github.com/Bwar/Nebula)或[__码云__](https://gitee.com/Bwar/Nebula)给个star，谢谢。Nebula不仅是一个框架，还提供了一系列基于这个框架的应用，目标是打造一个高性能分布式服务集群解决方案。


参考资料：
* 《UNIX网络编程》
* 《UNIX环境高级编程》
* [进程间传递文件描述符](https://pureage.info/2015/03/19/passing-file-descriptors.html)
* [linux网络编程之socket（十六）：通过UNIX域套接字传递描述符和 sendmsg/recvmsg 函数](https://blog.csdn.net/jnu_simba/article/details/9079627)

