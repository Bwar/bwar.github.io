---
layout: blog
article: true
category: C++
title:  基于OpenSSL的HTTPS通信C++实现
date:   2018-10-29 11:12:27
background-image: ../style/images/2018-06/bwar-article.png
tags:
- https
- ssl
---


&emsp;&emsp;HTTPS(全称：Hyper Text Transfer Protocol over Secure Socket Layer)，是以安全为目标的HTTP通道，简单讲是HTTP的安全版。即HTTP下加入SSL层，HTTPS的安全基础是SSL，因此加密的详细内容就需要SSL。[Nebula](https://github.com/Bwar/Nebula)是一个为开发者提供一个快速开发高并发网络服务程序或搭建高并发分布式服务集群的高性能事件驱动网络框架。Nebula作为通用网络框架提供HTTPS支持十分重要，Nebula既可用作https服务器，又可用作https客户端。本文将结合Nebula框架的https实现详细讲述基于openssl的SSL编程。如果觉得本文对你有用，帮忙到Nebula的[__Github__](https://github.com/Bwar/Nebula)或[__码云__](https://gitee.com/Bwar/Nebula)给个star，谢谢。Nebula不仅是一个框架，还提供了一系列基于这个框架的应用，目标是打造一个高性能分布式服务集群解决方案。Nebula的主要应用领域：即时通讯（成功应用于一款[IM](https://github.com/Bwar/Nebula/wiki/%E9%A6%96%E9%A1%B5)）、消息推送平台、数据实时分析计算([成功案例](https://github.com/Bwar/Nebio))等，Bwar还计划基于Nebula开发爬虫应用。

### SSL加密通信
&emsp;&emsp;HTTPS通信是在TCP通信层与HTTP应用层之间增加了SSL层，如果应用层不是HTTP协议也是可以使用SSL加密通信的，比如WebSocket协议WS的加上SSL层之后的WSS。Nebula框架可以通过更换Codec达到不修改代码变更通讯协议目的，Nebula增加SSL支持后，所有Nebula支持的通讯协议都有了SSL加密通讯支持，基于Nebula的业务代码无须做任何修改。

![https_communication]({{site.url}}/style/images/2019/https_communication.png)

&emsp;&emsp;Socket连接建立后的SSL连接建立过程：

![ssl_communication]({{site.url}}/style/images/2019/SSL.gif)

### OpenSSL API
&emsp;&emsp;OpenSSL的API很多，但并不是都会被使用到，如果需要查看某个API的详细使用方法可以阅读[API文档](https://www.openssl.org/docs/man1.1.0/ssl/)。
#### 初始化OpenSSL
&emsp;&emsp;OpenSSL在使用之前，必须进行相应的初始化工作。在建立SSL连接之前，要为Client和Server分别指定本次连接采用的协议及其版本，目前能够使用的协议版本包括SSLv2、SSLv3、SSLv2/v3和TLSv1.0。SSL连接若要正常建立，则要求Client和Server必须使用相互兼容的协议。
&emsp;&emsp;下面是Nebula框架SocketChannelSslImpl::SslInit()函数初始化OpenSSL的代码，根据OpenSSL的不同版本调用了不同的API进行初始化。
```C
#if OPENSSL_VERSION_NUMBER >= 0x10100003L

    if (OPENSSL_init_ssl(OPENSSL_INIT_LOAD_CONFIG, NULL) == 0)
    {
        pLogger->WriteLog(neb::Logger::ERROR, __FILE__, __LINE__, __FUNCTION__, "OPENSSL_init_ssl() failed!");
        return(ERR_SSL_INIT);
    }

    /*
     * OPENSSL_init_ssl() may leave errors in the error queue
     * while returning success
     */

    ERR_clear_error();

#else

    OPENSSL_config(NULL);

    SSL_library_init();         // 初始化SSL算法库函数( 加载要用到的算法 )，调用SSL函数之前必须调用此函数
    SSL_load_error_strings();   // 错误信息的初始化

    OpenSSL_add_all_algorithms();

#endif
```
#### 创建CTX
&emsp;&emsp;CTX是SSL会话环境，建立连接时使用不同的协议，其CTX也不一样。创建CTX的相关OpenSSL函数：

```C
//客户端、服务端都需要调用
SSL_CTX_new();                       //申请SSL会话环境

//若有验证对方证书的需求，则需调用
SSL_CTX_set_verify();                //指定证书验证方式
SSL_CTX_load_verify_location();      //为SSL会话环境加载本应用所信任的CA证书列表

//若有加载证书的需求，则需调用
int SSL_CTX_use_certificate_file();      //为SSL会话加载本应用的证书
int SSL_CTX_use_certificate_chain_file();//为SSL会话加载本应用的证书所属的证书链
int SSL_CTX_use_PrivateKey_file();       //为SSL会话加载本应用的私钥
int SSL_CTX_check_private_key();         //验证所加载的私钥和证书是否相匹配 
```

#### 创建SSL套接字
&emsp;&emsp;在创建SSL套接字之前要先创建Socket套接字，建立TCP连接。创建SSL套接字相关函数：

```C
SSL *SSl_new(SSL_CTX *ctx);          //创建一个SSL套接字
int SSL_set_fd(SSL *ssl, int fd);     //以读写模式绑定流套接字
int SSL_set_rfd(SSL *ssl, int fd);    //以只读模式绑定流套接字
int SSL_set_wfd(SSL *ssl, int fd);    //以只写模式绑定流套接字
```

#### 完成SSL握手
&emsp;&emsp;在这一步，我们需要在普通TCP连接的基础上，建立SSL连接。与普通流套接字建立连接的过程类似：Client使用函数SSL_connect()【类似于流套接字中用的connect()】发起握手，而Server使用函数SSL_ accept()【类似于流套接字中用的accept()】对握手进行响应，从而完成握手过程。两函数原型如下：
```C
int SSL_connect(SSL *ssl);
int SSL_accept(SSL *ssl);
```
&emsp;&emsp;握手过程完成之后，Client通常会要求Server发送证书信息，以便对Server进行鉴别。其实现会用到以下两个函数:
```C
X509 *SSL_get_peer_certificate(SSL *ssl);  //从SSL套接字中获取对方的证书信息
X509_NAME *X509_get_subject_name(X509 *a); //得到证书所用者的名字
```

#### 数据传输
&emsp;&emsp;经过前面的一系列过程后，就可以进行安全的数据传输了。在数据传输阶段，需要使用SSL_read( )和SSL_write( )来代替普通流套接字所使用的read( )和write( )函数，以此完成对SSL套接字的读写操作,两个新函数的原型分别如下：
```C
int SSL_read(SSL *ssl,void *buf,int num);            //从SSL套接字读取数据
int SSL_write(SSL *ssl,const void *buf,int num);     //向SSL套接字写入数据
```

#### 会话结束
&emsp;&emsp;当Client和Server之间的通信过程完成后，就使用以下函数来释放前面过程中申请的SSL资源：
```C
int SSL_shutdown(SSL *ssl);       //关闭SSL套接字
void SSl_free(SSL *ssl);          //释放SSL套接字
void SSL_CTX_free(SSL_CTX *ctx);  //释放SSL会话环境
```

### SSL 和 TLS
&emsp;&emsp;HTTPS 使用 SSL（Secure Socket Layer） 和 TLS（Transport LayerSecurity）这两个协议。
SSL 技术最初是由浏览器开发商网景通信公司率先倡导的，开发过 SSL3.0之前的版本。目前主导权已转移到 IETF（Internet Engineering Task Force，Internet 工程任务组）的手中。

&emsp;&emsp;IETF 以 SSL3.0 为基准，后又制定了 TLS1.0、TLS1.1 和 TLS1.2。TSL 是以SSL 为原型开发的协议，有时会统一称该协议为 SSL。当前主流的版本是SSL3.0 和 TLS1.0。

&emsp;&emsp;由于 SSL1.0 协议在设计之初被发现出了问题，就没有实际投入使用。SSL2.0 也被发现存在问题，所以很多浏览器直接废除了该协议版本。

### Nebula中的SSL通讯实现
&emsp;&emsp;Nebula框架同时支持SSL服务端应用和SSL客户端应用，对openssl的初始化只需要初始化一次即可（SslInit()只需调用一次）。Nebula框架的SSL相关代码（包括客户端和服务端的实现）都封装在[SocketChannelSslImpl](https://github.com/Bwar/Nebula/blob/master/src/channel/SocketChannelSslImpl.hpp)这个类中。Nebula的SSL通信是基于异步非阻塞的socket通信，并且不使用openssl的BIO（因为没有必要，代码还更复杂了）。

&emsp;&emsp;SocketChannelSslImpl是[SocketChannelImpl](https://github.com/Bwar/Nebula/blob/master/src/channel/SocketChannelImpl.hpp)的派生类，在SocketChannelImpl常规TCP通信之上增加了SSL通信层，两个类的调用几乎没有差异。SocketChannelSslImpl类声明如下：
```C++
class SocketChannelSslImpl : public SocketChannelImpl
{
public:
    SocketChannelSslImpl(SocketChannel* pSocketChannel, std::shared_ptr<NetLogger> pLogger, int iFd, uint32 ulSeq, ev_tstamp dKeepAlive = 0.0);
    virtual ~SocketChannelSslImpl();

    static int SslInit(std::shared_ptr<NetLogger> pLogger);
    static int SslServerCtxCreate(std::shared_ptr<NetLogger> pLogger);
    static int SslServerCertificate(std::shared_ptr<NetLogger> pLogger,
                const std::string& strCertFile, const std::string& strKeyFile);
    static void SslFree();

    int SslClientCtxCreate();
    int SslCreateConnection();
    int SslHandshake();
    int SslShutdown();

    virtual bool Init(E_CODEC_TYPE eCodecType, bool bIsClient = false) override;

    // 覆盖基类的Send()方法，实现非阻塞socket连接建立后继续建立SSL连接，并收发数据
    virtual E_CODEC_STATUS Send() override;      
    virtual E_CODEC_STATUS Send(int32 iCmd, uint32 uiSeq, const MsgBody& oMsgBody) override;
    virtual E_CODEC_STATUS Send(const HttpMsg& oHttpMsg, uint32 ulStepSeq) override;
    virtual E_CODEC_STATUS Recv(MsgHead& oMsgHead, MsgBody& oMsgBody) override;
    virtual E_CODEC_STATUS Recv(HttpMsg& oHttpMsg) override;
    virtual E_CODEC_STATUS Recv(MsgHead& oMsgHead, MsgBody& oMsgBody, HttpMsg& oHttpMsg) override;
    virtual bool Close() override;

protected:
    virtual int Write(CBuffer* pBuff, int& iErrno) override;
    virtual int Read(CBuffer* pBuff, int& iErrno) override;

private:
    E_SSL_CHANNEL_STATUS m_eSslChannelStatus;   //在基类m_ucChannelStatus通道状态基础上增加SSL通道状态
    bool m_bIsClientConnection;
    SSL* m_pSslConnection;

    static SSL_CTX* m_pServerSslCtx;    //当打开ssl选项编译，启动Nebula服务则自动创建
    static SSL_CTX* m_pClientSslCtx;    //默认为空，当打开ssl选项编译并且第一次发起了对其他SSL服务的连接时（比如访问一个https地址）创建
};
```
&emsp;&emsp;SocketChannelSslImpl类中带override关键字的方法都是覆盖基类SocketChannelImpl的同名方法，也是实现SSL通信与非SSL通信调用透明的关键。不带override关键字的方法都是SSL通信相关方法，这些方法里有openssl的函数调用。不带override的方法中有静态和非静态之分，静态方法在进程中只会被调用一次，与具体Channel对象无关。SocketChannel外部不需要调用非静态的ssl相关方法。

&emsp;&emsp;因为是非阻塞的socket，SSL_do_handshake()和SSL_write()、SSL_read()返回值并不完全能判断是否出错，还需要SSL_get_error()获取错误码。SSL_ERROR_WANT_READ和SSL_ERROR_WANT_WRITE都是正常的。

&emsp;&emsp;网上的大部分openssl例子程序是按顺序调用openssl函数简单实现同步ssl通信，在非阻塞IO应用中，ssl通信要复杂许多。SocketChannelSslImpl实现的是非阻塞的ssl通信，从该类的实现上看整个通信过程并非完全线性的。下面的SSL通信图更清晰地说明了Nebula框架中SSL通信是如何实现的：

![Nebula_ssl]({{site.url}}/style/images/2019/SSL_communication.jpg)

&emsp;&emsp;SocketChannelSslImpl中的静态方法在进程生命期内只需调用一次，也可以理解成SSL_CTX_new()、SSL_CTX_free()等方法只需调用一次。更进一步理解SSL_CTX结构体在进程内只需要创建一次（在Nebula中分别为Server和Client各创建一个）就可以为所有SSL连接所用；当然，为每个SSL连接创建独立的SSL_CTX也没问题（Nebula 0.4中实测过为每个Client创建独立的SSL_CTX），单一般不这么做，因为这样会消耗更多的内存资源，并且效率也会更低。

&emsp;&emsp;建立SSL连接时，客户端调用SSL_connect()，服务端调用SSL_accept()，许多openssl的demo都是这么用的。Nebula中用的是SSL_do_handshake()，这个方法同时适用于客户端和服务端，在兼具client和server功能的服务更适合用SSL_do_handshake()。注意调用SSL_do_handshake()前，如果是client端需要先调用SSL_set_connect_state()，如果是server端则需要先调用SSL_set_accept_state()。非阻塞IO中，SSL_do_handshake()可能需要调用多次才能完成握手，具体调用时机需根据SSL_get_error()获取错误码SSL_ERROR_WANT_READ和SSL_ERROR_WANT_WRITE判断需监听读事件还是写事件，在对应事件触发时再次调用SSL_do_handshake()。详细实现请参考SocketChannelSslImpl的Send和Recv方法。

&emsp;&emsp;关闭SSL连接时先调用SSL_shutdown()正常关闭SSL层连接（非阻塞IO中SSL_shutdown()亦可能需要调用多次）再调用SSL_free()释放SSL连接资源，最后关闭socket连接。SSL_CTX无须释放。整个SSL通信顺利完成，Nebula 0.4在开多个终端用shell脚本死循环调用curl简单压测中SSL client和SSL server功能一切正常:


```bash
while :
do 
     curl -v -k -H "Content-Type:application/json" -X POST -d '{"hello":"nebula ssl test"}' https://192.168.157.168:16003/test_ssl 
done
```

&emsp;&emsp;测试方法如下图：

![ssl_test]({{site.url}}/style/images/2019/https_test.jpg)

&emsp;&emsp;查看资源使用情况，SSL Server端的内存使用一直在增长，疑似有内存泄漏，不过pmap -d查看某一项anon内存达到近18MB时不再增长，说明可能不是内存泄漏，只是部分内存被openssl当作cache使用了。这个问题网上没找到解决办法。从struct ssl_ctx_st结构体定义发现端倪，再从nginx源码中发现了SSL_CTX_remove_session()，于是在SSL_free()之前加上SSL_CTX_remove_session()。session复用可以提高SSL通信效率，不过Nebula暂时不需要。

&emsp;&emsp;这种测试方法把NebulaInterface作为SSL服务端，NebulaLogic作为SSL客户端，同时完成了Nebula框架SSL服务端和客户端功能测试，简单的压力测试。Nebula框架的SSL通信测试通过，也可以投入生产应用，在后续应用中肯定还会继续完善。openssl真的难用，难怪被吐槽那么多，或许不久之后的Nebula版本将用其他ssl库替换掉openssl。

### 结束

&emsp;&emsp;加上SSL支持的Nebula框架测试通过，虽然不算太复杂，但过程还是蛮曲折，耗时也挺长。这里把Nebula使用openssl开发SSL通信分享出来，希望对准备使用openssl的开发者有用。如果觉得本文对你有用，别忘了到Nebula的[__Github__](https://github.com/Bwar/Nebula)或[__码云__](https://gitee.com/Bwar/Nebula)给个star，谢谢。


<br/>

参考资料：
* [基于OpenSSL实现的安全连接](http://www.cppblog.com/qinqing1984/archive/2014/04/11/206536.aspx)    
* [SSL API文档](https://www.openssl.org/docs/man1.1.0/ssl/)    
* [Https协议详解](https://blog.csdn.net/hellopaul8597/article/details/52327730)     
* [HTTPS是大势所趋？看腾讯专家通过Epoll+OpenSSL在高并发压测机器人中支持https](https://blog.csdn.net/wetest_tencent/article/details/53425198)
* [openssl 编程入门(含完整示例)](https://wenku.baidu.com/view/74575278ed630b1c58eeb517.html)
* [SSL连接建立过程分析](https://blog.csdn.net/jinhill/article/details/3615626) 
* [SSL socket 通讯详解](https://blog.csdn.net/pony_maggie/article/details/51315946)
* [HTTPS从原理到应用(三)：SSL/TLS协议](https://www.jianshu.com/p/c93612b3abac) 
* [SSL/TLS 握手优化详解](http://blog.jobbole.com/94332/) 
* [非阻塞/异步(epoll) openssl](http://www.cnblogs.com/dongfuye/p/4121066.html) 
* [两个基于openssl的https client例子](https://my.oschina.net/vincentwy/blog/620282?p=1)
* [OpenSSL编程初探1 --- 使用OpenSSL API建立SSL通信的一般流程简介](https://blog.csdn.net/howeverpf/article/details/18993945)
* [OpenSSL编程初探2 --- 关于证书文件的加载](https://blog.csdn.net/howeverpf/article/details/14108063)  

