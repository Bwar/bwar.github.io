---
layout: blog
article: true
category: C++
title:  Protobuf在Nebula框架中的应用
date:   2019-05-12 21:02:38
background-image: ../style/images/2018-06/bwar-article.png
tags:
- protobuf
- 协议
- Nebula
---

&emsp;&emsp;Protobuf应用广泛，尤其作为网络通讯协议最为普遍。本文将详细描述几个让人眼前一亮的protobuf协议设计，对准备应用或已经应用protobuf的开发者会有所启发，甚至可以直接拿过去用。 这里描述的协议设计被用于生产环境的即时通讯、埋点数据采集、消息推送、redis和mysql数据代理。

&emsp;&emsp;Bwar从2013年开始应用protobuf，2014年设计了用于mysql数据代理的protobuf协议，2015年设计了用于即时通讯的protobuf协议。高性能C++ IoC网络框架Nebula [https://github.com/Bwar/Nebula](https://github.com/Bwar/Nebula)把这几个protobuf协议设计应用到了极致。

### TCP通讯协议设计
&emsp;&emsp;本协议设计于2015年，用于一个生产环境的IM和埋点数据采集及实时分析，2016年又延伸发展了基于protobuf3的版本并用于开源网络框架[Nebula](https://github.com/Bwar/Nebula)。基于protobuf2和protobuf3的有较少差别，这里分开讲解两个版本的协议设计。

#### protobuf2.5版Msg
&emsp;&emsp;2015年尚无protobuf3的release版本，protobuf2版本的fixed32类型是固定占用4个字节的，非常适合用于网络通讯协议设计。Bwar设计用于IM系统的协议包括两个protobuf message：MsgHead和MsgBody，协议定义如下：

```C++
syntax = "proto2";

/**
 * @brief 消息头
 */
message MsgHead
{
    required fixed32 cmd = 1 ;                  ///< 命令字（压缩加密算法占高位1字节）
    required fixed32 msgbody_len = 2;           ///< 消息体长度（单个消息体长度不能超过65535即8KB）
    required fixed32 seq = 3;                   ///< 序列号
}

/**
 * @brief 消息体
 * @note 消息体主体是body，所有业务逻辑内容均放在body里。session_id和session用于接入层路由，
 * 两者只需要填充一个即可，首选session_id，当session_id用整型无法表达时才使用session。
 */
message MsgBody
{
    required bytes body         = 1;            ///< 消息体主体
    optional uint32 session_id  = 2;            ///< 会话ID（单聊消息为接收者uid，个人信息修改为uid，群聊消息为groupid，群管理为groupid）
    optional string session     = 3;            ///< 会话ID（当session_id用整型无法表达时使用）
    optional bytes additional   = 4;            ///< 接入层附加的数据（客户端无须理会）
}
```

&emsp;&emsp;解析收到的字节流时先解固定长度（15字节）的MsgHead（protobuf3.0之后的版本必须在cmd、msgbody_len、seq均不为0的情况下才是15字节），再通过MsgHead里的msgbody_len判断消息体是否接收完毕，若接收完毕则调用MsgBody.Parse()解析。MsgBody里的设计在下一节详细说明。

&emsp;&emsp;MsgHead在实际的项目应用中对应下面的消息头并可以相互转换：

```C++
#pragma pack(1)

/**
 * @brief 与客户端通信消息头
 */
struct tagClientMsgHead
{
    unsigned char version;                  ///< 协议版本号（1字节）
    unsigned char encript;                  ///< 压缩加密算法（1字节）
    unsigned short cmd;                     ///< 命令字/功能号（2字节）
    unsigned short checksum;                ///< 校验码（2字节）
    unsigned int body_len;                  ///< 消息体长度（4字节）
    unsigned int seq;                       ///< 序列号（4字节）
};

#pragma pack()
```

&emsp;&emsp;转换代码如下：
```C++
E_CODEC_STATUS ClientMsgCodec::Encode(const MsgHead& oMsgHead, const MsgBody& oMsgBody, loss::CBuffer* pBuff)
{
    tagClientMsgHead stClientMsgHead;
    stClientMsgHead.version = 1;        // version暂时无用
    stClientMsgHead.encript = (unsigned char)(oMsgHead.cmd() >> 24);
    stClientMsgHead.cmd = htons((unsigned short)(gc_uiCmdBit & oMsgHead.cmd()));
    stClientMsgHead.body_len = htonl((unsigned int)oMsgHead.msgbody_len());
    stClientMsgHead.seq = htonl(oMsgHead.seq());
    stClientMsgHead.checksum = htons((unsigned short)stClientMsgHead.checksum);
    ...
}

E_CODEC_STATUS ClientMsgCodec::Decode(loss::CBuffer* pBuff, MsgHead& oMsgHead, MsgBody& oMsgBody)
{
    LOG4_TRACE("%s() pBuff->ReadableBytes() = %u", __FUNCTION__, pBuff->ReadableBytes());
    size_t uiHeadSize = sizeof(tagClientMsgHead);
    if (pBuff->ReadableBytes() >= uiHeadSize)
    {
        tagClientMsgHead stClientMsgHead;
        int iReadIdx = pBuff->GetReadIndex();
        pBuff->Read(&stClientMsgHead, uiHeadSize);
        stClientMsgHead.cmd = ntohs(stClientMsgHead.cmd);
        stClientMsgHead.body_len = ntohl(stClientMsgHead.body_len);
        stClientMsgHead.seq = ntohl(stClientMsgHead.seq);
        stClientMsgHead.checksum = ntohs(stClientMsgHead.checksum);
        LOG4_TRACE("cmd %u, seq %u, len %u, pBuff->ReadableBytes() %u",
                        stClientMsgHead.cmd, stClientMsgHead.seq, stClientMsgHead.body_len,
                        pBuff->ReadableBytes());
        oMsgHead.set_cmd(((unsigned int)stClientMsgHead.encript << 24) | stClientMsgHead.cmd);
        oMsgHead.set_msgbody_len(stClientMsgHead.body_len);
        oMsgHead.set_seq(stClientMsgHead.seq);
        ...
    }
}
```

<br/>

#### protobuf3版Msg

&emsp;&emsp;protobuf3版的MsgHead和MsgBody从IM业务应用实践中发展而来，同时满足了埋点数据采集、实时计算、消息推送等业务需要，更为通用。正因其通用性和高扩展性，采用proactor模型的IoC网络框架[Nebula](https://github.com/Bwar/Nebula)才会选用这个协议，通过这个协议，框架层将网络通信工作从业务应用中完全独立出来，基于Nebula框架的应用开发者甚至可以不懂网络编程也能开发出高并发的分布式服务。

&emsp;&emsp;MsgHead和MsgBody的protobuf定义如下：

```C++
syntax = "proto3";

// import "google/protobuf/any.proto";

/**
 * @brief 消息头
 * @note MsgHead为固定15字节的头部，当MsgHead不等于15字节时，消息发送将出错。
 *       在proto2版本，MsgHead为15字节总是成立，cmd、seq、len都是required；
 *       但proto3版本，MsgHead为15字节则必须要求cmd、seq、len均不等于0，否则无法正确进行收发编解码。
 */
message MsgHead
{
    fixed32 cmd                = 1;           ///< 命令字（压缩加密算法占高位1字节）
    fixed32 seq                = 2;           ///< 序列号
    sfixed32 len               = 3;           ///< 消息体长度
}

/**
 * @brief 消息体
 * @note 消息体主体是data，所有业务逻辑内容均放在data里。req_target是请求目标，用于
 * 服务端接入路由，请求包必须填充。rsp_result是响应结果，响应包必须填充。
 */
message MsgBody
{
    oneof msg_type
    {
            Request req_target               = 1;                       ///< 请求目标（请求包必须填充）
            Response rsp_result              = 2;                       ///< 响应结果（响应包必须填充）
    }
    bytes data                  = 3;                    ///< 消息体主体
    bytes add_on                = 4;                    ///< 服务端接入层附加在请求包的数据（客户端无须理会）
    string trace_id             = 5;                    ///< for log trace

    message Request
    {
        uint32 route_id         = 1;                    ///< 路由ID
        string route            = 2;                    ///< 路由ID（当route_id用整型无法表达时使用）
    }

    message Response
    {
        int32 code            = 1;                     ///< 错误码
        bytes msg             = 2;                     ///< 错误信息
    }
}
```

&emsp;&emsp;MsgBody的data字段存储消息主体，任何自定义数据均可以二进制数据流方式写入到data。

&emsp;&emsp;msg_type用于标识该消息是请求还是响应（所有网络数据流都可归为请求或响应），如果是请求，则可以选择性填充Request里的route_id或route，如果填充了，则框架层无须解析应用层协议（也无法解析）就能自动根据路由ID转发，而无须应用层解开data里的内容再根据自定义逻辑转发。如果是响应，则定义了统一的错误标准，也为业务无关的错误处理提供方便。

&emsp;&emsp;add_on是附在长连接上的业务数据，框架并不会解析但会在每次转发消息时带上，可以为应用提供极其方便且强大的功能。比如，IM用户登录时客户端只发送用户ID和密码到服务端，服务端在登录校验通过后，将该用户的昵称、头像等信息通过框架提供的方法SetClientData()将数据附在服务端接入层该用户对应的长连接Channel上，之后所有从该连接过来的请求都会由框架层自动填充add_on字段，服务端的其他逻辑服务器只从data中得到自定义业务逻辑（比如聊天消息）数据，却可以从add_on中得到这个发送用户的信息。add_on的设计简化了应用开发逻辑，并降低了客户端与服务端传输的数据量。

&emsp;&emsp;trace_id用于分布式日志跟踪。分布式服务的错误定位是相当麻烦的，Nebula分布式服务解决方案提供了日志跟踪功能，协议里的trace_id字段的设计使得Nebula框架可以在完全不增加应用开发者额外工作的情况下（正常调用LOG4_INFO写日志而无须额外工作）实现所有标记着同一trace_id的日志发送到指定一台日志服务器，定义错误时跟单体服务那样登录一台服务器查看日志即可。比如，IM用户发送一条消息失败，在用户发送的消息到达服务端接入层时就被打上了trace_id标记，这个id会一直传递到逻辑层、存储层等，哪个环节发生了错误都可以从消息的发送、转发、处理路径上查到。

&emsp;&emsp;MsgHead和MsgBody的编解码实现见Nebula框架的[https://github.com/Bwar/Nebula/blob/master/src/codec/CodecProto.cpp](https://github.com/Bwar/Nebula/blob/master/src/codec/CodecProto.cpp)。

### Http通讯协议设计

&emsp;&emsp;上面的讲解的是protobuf应用于TCP数据流通信，接下来将描述protobuf在http通信上的应用。

&emsp;&emsp;在Web服务中通常会用Nginx做接入层的反向代理，经过Nginx转发到后续业务逻辑层的tomcat、apache或nginx上，接入层和业务逻辑层至少做了两次http协议解析，http协议是文本协议，传输数据量大解析速度慢。Nebula框架不是一个web服务器，但支持http协议，在只需提供http接口的应用场景（比如完全前后端分离的后端）基于Nebula的单进程http服务端并发量就可以是tomcat的数十倍。这一定程度上得益于Nebula框架在http通信上protobuf的应用。Nebula框架解析http文本协议并转化为HttpMsg在服务内部处理，应用开发者填充HttpMsg，接入层将响应的HttpMsg转换成http文本协议发回给请求方，不管服务端内部经过多少次中转，始终只有一次http协议的decode和一次http协议的encode。


```C++
syntax = "proto3";

message HttpMsg
{
        int32 type                             = 1;            ///< http_parser_type 请求或响应
        int32 http_major                       = 2;            ///< http大版本号
        int32 http_minor                       = 3;            ///< http小版本号
        int32 content_length                   = 4;            ///< 内容长度
        int32 method                           = 5;            ///< 请求方法
        int32 status_code                      = 6;            ///< 响应状态码
        int32 encoding                         = 7;            ///< 传输编码（只在encode时使用，当 Transfer-Encoding: chunked 时，用于标识chunk序号，0表示第一个chunk，依次递增）
        string url                             = 8;            ///< 地址
        map<string, string> headers            = 9;            ///< http头域
        bytes body                             = 10;           ///< 消息体（当 Transfer-Encoding: chunked 时，只存储一个chunk）
        map<string, string> params             = 11;           ///< GET方法参数，POST方法表单提交的参数
        Upgrade upgrade                        = 12;           ///< 升级协议
        float keep_alive                       = 13;           ///< keep alive time
        string path                            = 14;           ///< Http Decode时从url中解析出来，不需要人为填充（encode时不需要填）
        bool is_decoding                       = 15;           ///< 是否正在解码（true 正在解码， false 未解码或已完成解码）

        message Upgrade
        {
            bool is_upgrade             = 1;
            string protocol             = 2;
        }
}
```

&emsp;&emsp;HttpMsg的编解码实现见Nebula框架的[https://github.com/Bwar/Nebula/blob/master/src/codec/CodecHttp.cpp](https://github.com/Bwar/Nebula/blob/master/src/codec/CodecHttp.cpp)。

### 数据库代理服务协议设计

&emsp;&emsp;如果上面描述的protobuf在网络通信上应用算不错的话，那以下将protobuf用于数据代理上的协议设计则绝对是让人眼前一亮。

&emsp;&emsp;有的公司规定web服务不得直接访问MySQL数据库，甚至不允许在web逻辑层拼接SQL语句。如果有这种出于安全性考虑而做的限制，在web逻辑层后面再增加一层业务逻辑层成本未免太高了，那么解决办法应该是增加一层业务逻辑无关的代理服务层。这个代理服务层不是简单的转发SQL语句这么简单，因为web逻辑层可能不允许拼接SQL，由此引出我们这个用于数据库代理的protobuf协议设计。这个协议是将SQL逻辑融入整个协议之中，数据库代理层接收并解析这个协议后生成SQL语句或用binding方式到数据库去执行。数据库代理层只有协议解析和转化逻辑，无其他任何业务逻辑，业务逻辑还在web逻辑层，区别只在于从拼接SQL变成了填充protobuf协议。

```C++
syntax = "proto2";

package dbagent;

/**
 * @brief DB Agent消息
 */
message DbAgentMsg
{
    enum E_TYPE
    {
        UNDEFINE                      = 0;              ///< 未定义
        REQ_CONNECT                   = 1;              ///< 连接DB请求
        RSP_CONNECT                   = 2;              ///< 连接DB响应
        REQ_QUERY                     = 3;              ///< 执行SQL请求
        RSP_QUERY                     = 4;              ///< 执行SQL响应
        REQ_DISCONNECT                = 5;              ///< 关闭连接请求
        RSP_DISCONNECT                = 6;              ///< 关闭连接响应
        RSP_RECORD                    = 7;              ///< 结果集记录
        RSP_COMMON                    = 8;              ///< 通用响应（当请求不能被Server所认知时会做出这个回应）
        REQ_GET_CONNECT               = 9;              ///< 获取连接请求
        RSP_GET_CONNECT               = 10;             ///< 获取连接响应
    }

    required E_TYPE type                        = 1;    ///< 消息/操作 类型
    optional RequestConnect req_connect         = 2;    ///< 连接请求
    optional ResponseConnect rsp_connect        = 3;    ///< 连接响应
    optional RequestDisconnect req_disconnect   = 4;    ///< 关闭请求
    optional ResponseDisconnect rsp_disconnect  = 5;    ///< 关闭响应
    optional RequestQuery req_query             = 6;    ///< 执行SQL请求
    optional ResponseQuery rsp_query            = 7;    ///< 执行SQL响应
    optional ResponseRecord rsp_record          = 8;    ///< SELECT结果集记录
    optional ResponseCommon rsp_common          = 9;    ///< 通用响应
    optional RequestGetConnection req_get_conn  = 10;   ///< 获取连接请求
    optional ResponseGetConnection rsp_get_conn = 11;   ///< 获取连接响应
}

/**
 * @brief 连接请求
 */
message RequestConnect
{
    required string host        = 1;                    ///< DB所在服务器IP
    required int32  port        = 2;                    ///< DB端口
    required string user        = 3;                    ///< DB用户名
    required string password    = 4;                    ///< DB用户密码
    required string dbname      = 5;                    ///< DB库名
    required string charset     = 6;                    ///< DB字符集
}

/**
 * @brief 连接响应
 */
message ResponseConnect
{
    required int32 connect_id  = 1;                    ///< 连接ID （连接失败时，connect_id为0）
    optional int32 err_no       = 2;                   ///< 错误码 0 表示连接成功
    optional string err_msg     = 3;                   ///< 错误信息
}

/**
 * @brief 关闭连接请求
 */
message RequestDisconnect
{
    required int32 connect_id  = 1;                    ///< 连接ID （连接失败时，connect_id为0）
}

/**
 * @brief 关闭连接响应
 */
message ResponseDisconnect
{
    optional int32 err_no       = 2;                    ///< 错误码 0 表示连接成功
    optional string err_msg     = 3;                    ///< 错误信息
}

/**
 * @brief 执行SQL请求
 */
message RequestQuery
{
    required E_QUERY_TYPE query_type  = 1;              ///< 查询类型
    required string table_name        = 2;              ///< 表名
    repeated Field fields             = 3;              ///< 列类型
    repeated ConditionGroup conditions= 4;              ///< where条件组（由group_relation指定，若不指定则默认为AND关系）
    repeated string groupby_col       = 5;              ///< group by字段
    repeated OrderBy orderby_col      = 6;              ///< order by字段
    optional uint32 limit             = 7;              ///< 指定返回的行数的最大值  (limit 200)
    optional uint32 limit_from        = 8;              ///< 指定返回的第一行的偏移量 (limit 100, 200)
    optional ConditionGroup.E_RELATION group_relation = 9; ///< where条件组的关系,条件组之间有且只有一种关系（and或者or）
    optional int32 connect_id         = 10;             ///< 连接ID，有效连接ID（长连接，当connect后多次执行query可以使用connect_id）
    optional string bid               = 11;             ///< 业务ID，在CmdDbAgent.json配置文件中配置（短连接，每次执行query时连接，执行完后关闭连接）
    optional string password          = 12;             ///< 业务密码

    enum E_QUERY_TYPE                                   ///< 查询类型
    {
        SELECT                        = 0;              ///< select查询
        INSERT                        = 1;              ///< insert插入
        INSERT_IGNORE                 = 2;              ///< insert ignore插入，若存在则放弃
        UPDATE                        = 3;              ///< update更新
        REPLACE                       = 4;              ///< replace覆盖插入
        DELETE                        = 5;              ///< delete删除
    }

    enum E_COL_TYPE                                     ///< 列类型
    {
        STRING                        = 0;              ///< char, varchar, text, datetime, timestamp等
        INT                           = 1;              ///< int
        BIGINT                        = 2;              ///< bigint
        FLOAT                         = 3;              ///< float
        DOUBLE                        = 4;              ///< double
    }

    message Field                                       ///< 字段
    {
        required string col_name      = 1;              ///< 列名
        required E_COL_TYPE col_type  = 2;              ///< 列类型
        required bytes col_value      = 3;              ///< 列值
        optional string col_as        = 4;              ///< as列名
    }

    message Condition                                   ///< where条件
    {
        required E_RELATION relation  = 1;              ///< 关系（=, !=, >, <, >=, <= 等）
        required E_COL_TYPE col_type  = 2;              ///< 列类型
        required string col_name      = 3;              ///< 列名
        repeated bytes col_values     = 4;              ///< 列值（当且仅当relation为IN时值的个数大于1有效）
        optional string col_name_right= 5;              ///< 关系右边列名（用于where col1=col2这种情况）
        enum E_RELATION
        {
            EQ                        = 0;              ///< 等于=
            NE                        = 1;              ///< 不等于!=
            GT                        = 2;              ///< 大于>
            LT                        = 3;              ///< 小于<
            GE                        = 4;              ///< 大于等于>=
            LE                        = 5;              ///< 小于等于<=
            LIKE                      = 6;              ///< like
            IN                        = 7;              ///< in (1, 2, 3, 4, 5)
        }
    }

    message ConditionGroup                              ///< where条件组合
    {
        required E_RELATION relation     = 1;           ///< 条件之间的关系，一个ConditionGroup里的所有Condition之间有且只有一种关系（and或者or）
        repeated Condition condition     = 2;           ///< 条件
        enum E_RELATION
        {
            AND                        = 0;             ///< and且
            OR                         = 1;             ///< or或
        }
    }

    message OrderBy
    {
        required E_RELATION relation    = 1;            ///< 降序或升序
        required string col_name        = 2;            ///< 列名
        enum E_RELATION
        {
            ASC                         = 0;
            DESC                        = 1;
        }
    }
}

/**
 * @brief 执行SQL响应
 */
message ResponseQuery
{
    required uint32 seq         = 1;                    ///< 数据包序列号（SELECT结果集会分包返回，只有一个包的情况或已到达最后一个包则seq=0xFFFFFFFF）
    required int32 err_no       = 2;                    ///< 错误码，0 表示执行成功
    optional string err_msg     = 3;                    ///< 错误信息
    optional uint64 insert_id   = 4;                    ///< mysql_insert_id()获取的值（视执行的SQL语句而定，不一定存在）
    repeated bytes dict         = 5;                    ///< 结果集字典（视执行的SQL语句而定，不一定存在）
}

/**
 * @brief SELECT语句返回结果集的一条记录
 */
message ResponseRecord
{
    required uint32 seq         = 1;                    ///< 数据包序列号（SELECT结果集会分包返回，已到达最后一个包则seq=0xFFFFFFFF）
    repeated bytes field        = 2;                    ///< 数据集记录的字段
}

/**
 * @brief 常规响应
 */
message ResponseCommon
{
    optional int32 err_no       = 1;                    ///< 错误码 0 表示连接成功
    optional string err_msg     = 2;                    ///< 错误信息
}

/**
 * @brief 获取连接请求
 */
message RequestGetConnection
{
    required string bid         = 1;                    ///< 业务ID，在dbproxy配置文件中配置
    required string password    = 2;                    ///< 业务密码
}

/**
 * @brief 获取连接响应
 */
message ResponseGetConnection
{
    required int32 connect_id   = 1;                    ///< 连接ID，有效连接ID，否则执行失败
    optional int32 err_no       = 2;                   ///< 错误码 0 表示连接成功
    optional string err_msg     = 3;                   ///< 错误信息
}
```

&emsp;&emsp;基于这个数据库操作协议开发的数据库代理层完全解决了web逻辑层不允许直接访问数据库也不允许拼接SQL语句的问题，而且几乎没有增加开发代价。另外，基于这个协议的数据库代理天然防止SQL注入（在代理层校验field_name，并且mysql_escape_string(filed_value)），虽然防SQL注入应是应用层的责任，但多了数据代理这层保障也是好事。

&emsp;&emsp;这个协议只支持简单SQL，不支持联合查询、子查询，也不支持存储过程，如果需要支持的话协议会更复杂。在Bwar所负责过的业务里，基本都禁止数据库联合查询之类，只把数据库当存储用，不把逻辑写到SQL语句里，所以这个协议满足大部分业务需要。

&emsp;&emsp;这一节只说明数据库代理协议，下一节将从数据库代理协议延伸并提供协议代码讲解。

### Redis和MySQL数据代理协议设计

&emsp;&emsp;大部分后台应用只有MySQL是不够的，往往还需要缓存，经常会用Redis来做数据缓存。用缓存意味着数据至少需要同时写到Redis和MySQL，又或者在未命中缓存时从MySQL中读取到的数据需要回写到Redis，这些通常都是由业务逻辑层来做的。也有例外，Nebula提供的分布式解决方案是由数据代理层来做的，业务逻辑层只需向数据代理层发送一个protobuf协议数据，数据代理层就会完成Redis和MySQL双写或缓存未命中时的自动回写（暂且不探讨数据一致性问题）。数据代理层来做这些工作是为了减少业务逻辑层的复杂度，提高开发效率。既然是为了提高开发效率，就得让业务逻辑层低于原来同时操作Redis和MySQL的开发量。Nebula提供的[NebulaMydis](https://github.com/Bwar/NebulaMydis)就是这样一个让原来同时操作Redis和MySQL的开发量（假设是2）降到1.2左右。

&emsp;&emsp;这个同时操作Redis和MySQL的数据代理协议如下：

```C++
syntax = "proto3";

package neb;

message Mydis
{
    uint32 section_factor       = 1;
    RedisOperate redis_operate  = 2;
    DbOperate db_operate        = 3;

    message RedisOperate
    {
        bytes key_name           = 1;
        string redis_cmd_read    = 2;
        string redis_cmd_write   = 3;
        OPERATE_TYPE op_type     = 4;
        repeated Field fields    = 5;
        int32 key_ttl            = 6;
        int32 redis_structure    = 7;      ///< redis数据类型
        int32 data_purpose       = 8;      ///< 数据用途
        bytes hash_key           = 9;      ///< 可选hash key，当has_hash_key()时用hash_key来计算hash值，否则用key_name来计算hash值

        enum OPERATE_TYPE
        {
            T_READ  = 0;
            T_WRITE = 1;
        }
    }

    message DbOperate
    {
        E_QUERY_TYPE query_type                   = 1;         ///< 查询类型
        string table_name                         = 2;         ///< 表名
        repeated Field fields                     = 3;         ///< 列类型
        repeated ConditionGroup conditions        = 4;         ///< where条件组（由group_relation指定，若不指定则默认为AND关系）
        repeated string groupby_col               = 5;         ///< group by字段
        repeated OrderBy orderby_col              = 6;         ///< order by字段
        ConditionGroup.E_RELATION group_relation  = 7;         ///< where条件组的关系,条件组之间有且只有一种关系（and或者or）
        uint32 limit                              = 8;         ///< 指定返回的行数的最大值  (limit 200)
        uint32 limit_from                         = 9;         ///< 指定返回的第一行的偏移量 (limit 100, 200)
        uint32 mod_factor                         = 10;        ///< 分表取模因子，当这个字段没有时使用section_factor

        enum E_QUERY_TYPE                                      ///< 查询类型
        {
            SELECT                        = 0;              ///< select查询
            INSERT                        = 1;              ///< insert插入
            INSERT_IGNORE                 = 2;              ///< insert ignore插入，若存在则放弃
            UPDATE                        = 3;              ///< update更新
            REPLACE                       = 4;              ///< replace覆盖插入
            DELETE                        = 5;              ///< delete删除
        }

        message Condition                                         ///< where条件
        {
            E_RELATION relation                  = 1;              ///< 关系（=, !=, >, <, >=, <= 等）
            E_COL_TYPE col_type                  = 2;              ///< 列类型
            string col_name                      = 3;              ///< 列名
            repeated bytes col_values            = 4;              ///< 列值（当且仅当relation为IN时值的个数大于1有效）
            string col_name_right                = 5;              ///< 关系右边列名（用于where col1=col2这种情况）
            enum E_RELATION
            {
                EQ                        = 0;              ///< 等于=
                NE                        = 1;              ///< 不等于!=
                GT                        = 2;              ///< 大于>
                LT                        = 3;              ///< 小于<
                GE                        = 4;              ///< 大于等于>=
                LE                        = 5;              ///< 小于等于<=
                LIKE                      = 6;              ///< like
                IN                        = 7;              ///< in (1, 2, 3, 4, 5)
            }
        }

        message ConditionGroup                              ///< where条件组合
        {
            E_RELATION relation                      = 1;           ///< 条件之间的关系，一个ConditionGroup里的所有Condition之间有且只有一种关系（and或者or）
            repeated Condition condition             = 2;           ///< 条件
            enum E_RELATION
            {
                AND                        = 0;             ///< and且
                OR                         = 1;             ///< or或
            }
        }

        message OrderBy
        {
            E_RELATION relation                      = 1;            ///< 降序或升序
            string col_name                          = 2;            ///< 列名
            enum E_RELATION
            {
                ASC                         = 0;
                DESC                        = 1;
            }
        }
    }
}

enum E_COL_TYPE                               ///< 列类型
{
    STRING                        = 0;        ///< char, varchar, text, datetime, timestamp等
    INT                           = 1;        ///< int
    BIGINT                        = 2;        ///< bigint
    FLOAT                         = 3;        ///< float
    DOUBLE                        = 4;        ///< double
}

message Record
{
    repeated Field field_info     = 1;        ///< value data
}

message Field                                  ///< 字段
{
    string col_name      = 1;         ///< 列名
    E_COL_TYPE col_type  = 2;         ///< 列类型
    bytes col_value      = 3;         ///< 列值
    string col_as        = 4;         ///< as列名
}



/**
 * @brief 查询结果
 * @note 适用于Redis返回和MySQL返回，当totalcount与curcount相等时表明数据已接收完毕，
 * 否则表示数据尚未接收完，剩余的数据会在后续数据包继续返回。
 */
message Result
{
    int32 err_no                                   = 1;
    bytes err_msg                                  = 2;
    int32 total_count                              = 3;
    int32 current_count                            = 4;
    repeated Record record_data                    = 5;
    int32 from                                     = 6;  ///< 数据来源 E_RESULT_FROM
    DataLocate locate                              = 7;  ///< 仅在DataProxy使用
    enum E_RESULT_FROM
    {
        FROM_DB                     = 0;
        FROM_REDIS                  = 1;
    }
    message DataLocate
    {
        uint32 section_from    = 1;
        uint32 section_to      = 2;  ///< 数据所在分段，section_from < MemOperate.section_factor <= section_to
        uint32 hash            = 3;  ///< 用于做分布的hash值（取模运算时，为取模后的结果）
        uint32 divisor         = 4;  ///< 取模运算的除数（一致性hash时不需要）
    }
}
```

&emsp;&emsp;这个协议分了Redis和MySQL两部分数据，看似业务逻辑层把一份数据填充了两份并没有降低多少开发量，实际上这两部分数据有许多是可共用的，只要提供一个填充类就可以大幅降低协议填充开发量。为简化协议填充，Nebula提供了几个类：[同时填充Redis和MySQL数据](https://github.com/Bwar/Nebula/blob/master/src/mydis/MydisOperator.hpp)、[只填充Redis](https://github.com/Bwar/Nebula/blob/master/src/mydis/RedisOperator.hpp)、[只填充MySQL](https://github.com/Bwar/Nebula/blob/master/src/mydis/DbOperator.hpp)。

&emsp;&emsp;从Mydis协议的MySQL部分如何生成SQL语句请参考[NebulaDbAgent](https://github.com/Bwar/NebulaDbAgent)，核心代码头文件如下：

```C++
namespace dbagent
{

const int gc_iMaxBeatTimeInterval = 30;
const int gc_iMaxColValueSize = 65535;

struct tagConnection
{
    CMysqlDbi* pDbi;
    time_t ullBeatTime;
    int iQueryPermit;
    int iTimeout;

    tagConnection() : pDbi(NULL), ullBeatTime(0), iQueryPermit(0), iTimeout(0)
    {
    }

    ~tagConnection()
    {
        if (pDbi != NULL)
        {
            delete pDbi;
            pDbi = NULL;
        }
    }
};

class CmdExecSql : public neb::Cmd, public neb::DynamicCreator<CmdExecSql, int32>
{
public:
    CmdExecSql(int32 iCmd);
    virtual ~CmdExecSql();

    virtual bool Init();

    virtual bool AnyMessage(
                    std::shared_ptr<neb::SocketChannel> pChannel,
                    const MsgHead& oMsgHead,
                    const MsgBody& oMsgBody);

protected:
    bool GetDbConnection(const neb::Mydis& oQuery, CMysqlDbi** ppMasterDbi, CMysqlDbi** ppSlaveDbi);
    bool FetchOrEstablishConnection(neb::Mydis::DbOperate::E_QUERY_TYPE eQueryType,
                    const std::string& strMasterIdentify, const std::string& strSlaveIdentify,
                    const neb::CJsonObject& oInstanceConf, CMysqlDbi** ppMasterDbi, CMysqlDbi** ppSlaveDbi);
    std::string GetFullTableName(const std::string& strTableName, uint32 uiFactor);

    int ConnectDb(const neb::CJsonObject& oInstanceConf, CMysqlDbi* pDbi, bool bIsMaster = true);
    int Query(const neb::Mydis& oQuery, CMysqlDbi* pDbi);
    void CheckConnection(); //检查连接是否已超时
    void Response(int iErrno, const std::string& strErrMsg);
    bool Response(const neb::Result& oRsp);

    bool CreateSql(const neb::Mydis& oQuery, CMysqlDbi* pDbi, std::string& strSql);
    bool CreateSelect(const neb::Mydis& oQuery, std::string& strSql);
    bool CreateInsert(const neb::Mydis& oQuery, CMysqlDbi* pDbi, std::string& strSql);
    bool CreateUpdate(const neb::Mydis& oQuery, CMysqlDbi* pDbi, std::string& strSql);
    bool CreateDelete(const neb::Mydis& oQuery, std::string& strSql);
    bool CreateCondition(const neb::Mydis::DbOperate::Condition& oCondition, CMysqlDbi* pDbi, std::string& strCondition);
    bool CreateConditionGroup(const neb::Mydis& oQuery, CMysqlDbi* pDbi, std::string& strCondition);
    bool CreateGroupBy(const neb::Mydis& oQuery, std::string& strGroupBy);
    bool CreateOrderBy(const neb::Mydis& oQuery, std::string& strOrderBy);
    bool CreateLimit(const neb::Mydis& oQuery, std::string& strLimit);
    bool CheckColName(const std::string& strColName);

private:
    std::shared_ptr<neb::SocketChannel> m_pChannel;
    MsgHead m_oInMsgHead;
    MsgBody m_oInMsgBody;
    int m_iConnectionTimeout;   //空闲连接超时（单位秒）
    char* m_szColValue;         //字段值
    neb::CJsonObject m_oDbConf;
    uint32 m_uiSectionFrom;
    uint32 m_uiSectionTo;
    uint32 m_uiHash;
    uint32 m_uiDivisor;
    std::map<std::string, std::set<uint32> > m_mapFactorSection; //分段因子区间配置，key为因子类型
    std::map<std::string, neb::CJsonObject*> m_mapDbInstanceInfo;  //数据库配置信息key为("%u:%u:%u", uiDataType, uiFactor, uiFactorSection)
    std::map<std::string, tagConnection*> m_mapDbiPool;     //数据库连接池，key为identify（如：192.168.18.22:3306）
};

} // namespace dbagent
```

&emsp;&emsp;整个mydis数据协议是如何解析如何使用，如何做Redis和MySQL的数据双写、缓存数据回写等不在本文讨论范围，如有兴趣可以阅读[NebulaMydis](https://github.com/Bwar/NebulaMydis)源码，也可以联系Bwar。


### 结语

&emsp;&emsp;Protobuf用得合适用得好可以解决许多问题，可以提高开发效率，也可以提高运行效率，以上就是Bwar多年应用protobuf的小结，没有任何藏私，文中列出的协议都可以在开源项目[Nebula](https://github.com/Bwar/Nebula)的这个路径[https://github.com/Bwar/Nebula/tree/master/proto](https://github.com/Bwar/Nebula/tree/master/proto)找到。

&emsp;&emsp;开发Nebula框架目的是致力于提供一种基于C\+\+快速构建高性能的分布式服务。如果觉得本文对你有用，别忘了到Nebula的[__Github__](https://github.com/Bwar/Nebula)或[__码云__](https://gitee.com/Bwar/Nebula)给个star，谢谢。


