---
layout: blog
article: true
category: C++
title:  集群协调服务选举实现
date:   2019-01-13 15:42:33
background-image: ../style/images/2018-06/bwar-article.png
tags:
- ZooKeeper
- 协调服务
- Nebula
- 选举
- Paxos
---


&emsp;&emsp;一个分布式服务集群管理通常需要一个协调服务，提供服务注册、服务发现、配置管理、组服务等功能，而协调服务自身应是一个高可用的服务集群，ZooKeeper是广泛应用且众所周知的协调服务。协调服务自身的高可用需要选举算法来支撑，本文将讲述选举原理并以分布式服务集群[NebulaBootstrap](https://github.com/Bwar/NebulaBootstrap)的协调服务[NebulaBeacon](https://github.com/Bwar/NebulaBeacon)为例详细说明协调服务的选举实现。
&emsp;&emsp;为什么要选NebulaBeacon来说明协调服务的选举实现？一方面是我没有读过Zookeeper的代码，更重要的另一方面是NebulaBeacon的选举实现只有两百多行代码，简单精炼，很容易讲清楚。基于高性能C++网络框架[Nebula](https://github.com/Bwar/Nebula)实现的分布式服务集群[NebulaBootstrap](https://github.com/Bwar/NebulaBootstrap)是一种用C++快速构建高性能分布式服务的解决方案。为什么要实现自己的协调服务而不直接用Zookeeper？想造个C++的轮子，整个集群都是C++服务，因为选了ZooKeeper而需要部署一套Java环境，配置也跟其他服务不是一个体系，实在不是一个好的选择。Spring Cloud有Eureka，NebulaBootstrap有NebulaBeacon，未来NebulaBootstrap会支持ZooKeeper，不过暂无时间表，还是首推NebulaBeacon。

### 选举算法选择
&emsp;&emsp;__Paxos算法__ 和 __ZooKeeper ZAB协议__ 是两种较广为人知的选举算法。ZAB协议主要用于构建一个高可用的分布式数据主备系统，例如ZooKeeper，而Paxos算法则是用于构建一个分布式的一致性状态机系统。也有很多应用程序采用自己设计的简单的选举算法，这类型简单的选举算法通常依赖计算机自身因素作为选举因子，比如IP地址、CPU核数、内存大小、自定义序列号等。

&emsp;&emsp;Paxos规定了四种角色（Proposer，Acceptor，Learner，以及Client）和两个阶段（Promise和Accept）。
&emsp;&emsp;ZAB服务具有四种状态：LOOKING、FOLLOWING、LEADING、OBSERVING。
&emsp;&emsp;NebulaBeacon是高可用分布式系统的协调服务，采用ZAP协议更为合适，不过ZAP协议还是稍显复杂了，NebulaBeacon的选举算法实现基于节点的IP地址标识，选举速度快，实现十分简单。

<br/>

### 选举相关数据结构
&emsp;&emsp;NebulaBeacon的选举相关数据结构非常简单：

```C++
const uint32 SessionOnlineNodes::mc_uiLeader = 0x80000000;   ///< uint32最高位为1表示leader
const uint32 SessionOnlineNodes::mc_uiAlive  = 0x00000007;   ///< 最近三次心跳任意一次成功则认为在线
std::map<std::string, uint32> m_mapBeacon;                   ///< Key为节点标识，值为在线心跳及是否为leader标识
```

&emsp;&emsp;如上数据结构m_mapBeacon保存了Beacon集群各Beacon节点信息，以Beacon节点的IP地址标识为key排序，每次遍历均从头开始，满足条件（1&&2 或者 1&&3）则标识为Leader：1. 节点在线；2. 已经成为Leader； 3. 整个列表中不存在在线的Leader，而节点处于在线节点列表的首位。

### Beacon选举流程
&emsp;&emsp;Beacon选举基于节点IP地址标识，实现非常简单且高效。

```json
"beacon":["192.168.1.11:16000", "192.168.1.12:16000"]
```

&emsp;&emsp;进程启动时首先检查[Beacon集群配置](https://github.com/Bwar/NebulaBeacon/blob/master/conf/NebulaBeacon.json)，若未配置其他Beacon节点信息，则默认只有一个Beacon节点，此时该节点在启动时自动成为Leader节点。否则，向其他Beacon节点发送一个心跳消息，等待定时器回调检查并选举出Leader节点。选举流程如下图：

![Beacon选举流程](../style/images/2019/beacon_election.png)

&emsp;&emsp;检查是否在线就是通过检查两次定时器回调之间是否收到了其他Beacon节点的心跳消息。对m_mapBeacon的遍历检查判断节点在线情况，对已离线的Leader节点置为离线状态，若当前节点应成为Leader节点则成为Leader节点。

### Beacon节点间选举通信

&emsp;&emsp;Beacon节点间的选举通信与节点心跳合为一体，这样做的好处是当leader节点不可用时，fllower节点立刻可以成为leader节点，选举过程只需每个fllower节点遍历自己内存中各Beacon节点的心跳信息即可，无须在发现leader不在线才发起选举，更快和更好地保障集群的高可用性。

![Beacon间心跳](../style/images/2019/beacon_beat.jpg)

&emsp;&emsp;Beacon节点心跳信息带上了leader节点作为协调服务产生的新数据，fllower节点在接收心跳的同时完成了数据同步，保障任意一个fllower成为leader时已获得集群所有需协调的信息并可随时切换为leader。除定时器触发的心跳带上协调服务产生的新数据之外，leader节点产生新数据的同时会立刻向fllower发送心跳。

### Beacon选举实现

&emsp;&emsp;Beacon心跳协议proto：
```proto
/**
 * @brief BEACON节点间心跳
 */
message Election
{
    int32 is_leader                  = 1;    ///< 是否主节点
    uint32 last_node_id              = 2;    ///< 上一个生成的节点ID
    repeated uint32 added_node_id    = 3;    ///< 新增已使用的节点ID
    repeated uint32 removed_node_id  = 4;    ///< 删除已废弃的节点ID
}
```

&emsp;&emsp;检查Beacon配置，若只有一个Beacon节点则自动成为Leader：

```C++
void SessionOnlineNodes::InitElection(const neb::CJsonObject& oBeacon)
{
    neb::CJsonObject oBeaconList = oBeacon;
    for (int i = 0; i < oBeaconList.GetArraySize(); ++i)
    {
        m_mapBeacon.insert(std::make_pair(oBeaconList(i) + ".1", 0));
    }
    if (m_mapBeacon.size() == 0)
    {
        m_bIsLeader = true;
    }
    else if (m_mapBeacon.size() == 1
            && GetNodeIdentify() == m_mapBeacon.begin()->first)
    {
        m_bIsLeader = true;
    }
    else
    {
        SendBeaconBeat();
    }
}
```

&emsp;&emsp;发送Beacon心跳：

```C++
void SessionOnlineNodes::SendBeaconBeat()
{
    LOG4_TRACE("");
    MsgBody oMsgBody;
    Election oElection;
    if (m_bIsLeader)
    {
        oElection.set_is_leader(1);
        oElection.set_last_node_id(m_unLastNodeId);
        for (auto it = m_setAddedNodeId.begin(); it != m_setAddedNodeId.end(); ++it)
        {
            oElection.add_added_node_id(*it);
        }
        for (auto it = m_setRemovedNodeId.begin(); it != m_setRemovedNodeId.end(); ++it)
        {
            oElection.add_removed_node_id(*it);
        }
    }
    else
    {
        oElection.set_is_leader(0);
    }
    m_setAddedNodeId.clear();
    m_setRemovedNodeId.clear();
    oMsgBody.set_data(oElection.SerializeAsString());

    for (auto iter = m_mapBeacon.begin(); iter != m_mapBeacon.end(); ++iter)
    {
        if (GetNodeIdentify() != iter->first)
        {
            SendTo(iter->first, neb::CMD_REQ_LEADER_ELECTION, GetSequence(), oMsgBody);
        }
    }
}
```

&emsp;&emsp;接收Beacon心跳：

```C++
void SessionOnlineNodes::AddBeaconBeat(const std::string& strNodeIdentify, const Election& oElection)
{
    if (!m_bIsLeader)
    {
        if (oElection.last_node_id() > 0)
        {
            m_unLastNodeId = oElection.last_node_id();
        }
        for (int32 i = 0; i < oElection.added_node_id_size(); ++i)
        {
            m_setNodeId.insert(oElection.added_node_id(i));
        }
        for (int32 j = 0; j < oElection.removed_node_id_size(); ++j)
        {
            m_setNodeId.erase(m_setNodeId.find(oElection.removed_node_id(j)));
        }
    }

    auto iter = m_mapBeacon.find(strNodeIdentify);
    if (iter == m_mapBeacon.end())
    {
        uint32 uiBeaconAttr = 1;
        if (oElection.is_leader() != 0)
        {
            uiBeaconAttr |= mc_uiLeader;
        }
        m_mapBeacon.insert(std::make_pair(strNodeIdentify, uiBeaconAttr));
    }
    else
    {
        iter->second |= 1;
        if (oElection.is_leader() != 0)
        {
            iter->second |= mc_uiLeader;
        }
    }
}
```

&emsp;&emsp;检查在线leader，成为leader：

```C++
void SessionOnlineNodes::CheckLeader()
{
    LOG4_TRACE("");
    std::string strLeader;
    for (auto iter = m_mapBeacon.begin(); iter != m_mapBeacon.end(); ++iter)
    {
        if (mc_uiAlive & iter->second)
        {
            if (mc_uiLeader & iter->second)
            {
                strLeader = iter->first;
            }
            else if (strLeader.size() == 0)
            {
                strLeader = iter->first;
            }
        }
        else
        {
            iter->second &= (~mc_uiLeader);
        }
        uint32 uiLeaderBit = mc_uiLeader & iter->second;
        iter->second = ((iter->second << 1) & mc_uiAlive) | uiLeaderBit;
        if (iter->first == GetNodeIdentify())
        {
            iter->second |= 1;
        }
    }

    if (strLeader == GetNodeIdentify())
    {
        m_bIsLeader = true;
    }
}
```

### Beacon节点切换leader

&emsp;&emsp;通过Nebula集群的命令行管理工具[nebcli](https://github.com/Bwar/Nebcli)可以很方便的查看Beacon节点状态，nebcli的使用说明见Nebcli项目的README。下面启动三个Beacon节点，并反复kill掉Beacon进程和重启，查看leader节点的切换情况。

&emsp;&emsp;启动三个beacon节点：

```
nebcli): show beacon
node                        is_leader       is_online
192.168.157.176:16000.1     yes             yes
192.168.157.176:17000.1     no              yes
192.168.157.176:18000.1     no              yes
```

&emsp;&emsp;kill掉leader节点：

```
nebcli): show beacon
node                        is_leader       is_online
192.168.157.176:16000.1     no              no
192.168.157.176:17000.1     yes             yes
192.168.157.176:18000.1     no              yes
```

&emsp;&emsp;kill掉fllower节点：

```
nebcli): show beacon
node                        is_leader       is_online
192.168.157.176:16000.1     no              no
192.168.157.176:17000.1     yes             yes
192.168.157.176:18000.1     no              no
```

&emsp;&emsp;重启被kill掉的两个节点：

```
nebcli): show beacon
node                        is_leader       is_online
192.168.157.176:16000.1     no              yes
192.168.157.176:17000.1     yes             yes
192.168.157.176:18000.1     no              yes
```

&emsp;&emsp;fllower节点在原leader节点不可用后成为leader节点，且只要不宕机则一直会是leader节点，即使原leader节点重新变为可用状态也不会再次切换。

### 结束

&emsp;&emsp;开发Nebula框架目的是致力于提供一种基于C\+\+快速构建高性能的分布式服务。如果觉得本文对你有用，别忘了到Nebula的[__Github__](https://github.com/Bwar/Nebula)或[__码云__](https://gitee.com/Bwar/Nebula)给个star，谢谢。

