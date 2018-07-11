---
layout: blog
article: true
category: data
title:  Nebio：数据统计分析(二)—数据埋点统计库表结构
date:   2018-07-08 14:47:38
tags:
- data
- analyse
---

## db_nebio_conf 配置库
nebio相关配置信息存储。
### tb_app 应用配置表
&emsp;&emsp;配置应用信息。

| 字段名  | 类型  | 键 | 字段描述  | 示例
| ------------ | ------------ | ------------ | ------------ | ------------ |
| app_id | int(11) unsigned | PRI | 应用ID | 100001 |
| app_key | varchar(64) | INDEX | 应用标识 | dwehgiiwae |
| app_name | varchar(10) | | 应用名称 | 官网 |
| create_time | timestamp | | 创建时间 | 2018-07-08 14:20:00 |
| update_time | timestamp | | 更新时间 | 2018-07-09 20:10:00 |

<br/>

### tb_event 事件配置表
&emsp;&emsp;配置事件信息。

| 字段名  | 类型  | 键 | 字段描述  | 示例
| ------------ | ------------ | ------------ | ------------ | ------------ |
| app_id | int(11) unsigned | PRI | 应用ID | 100001 |
| event_id | varchar(64) | PRI | 事件ID | home_page |
| event_name | varchar(16) | | 事件名称 | 首页 |
| event_desc | varchar(64) | | 事件描述 | 官网首页 |
| event_type | varchar(16) | | 事件类型 | page |
| create_time | timestamp | | 创建时间 | 2018-07-08 14:20:00 |
| update_time | timestamp | | 更新时间 | 2018-07-09 20:10:00 |

<br/>

### tb_page 页面名称配置表
&emsp;&emsp;配置页面信息。

| 字段名  | 类型  | 键 | 字段描述  | 示例
| ------------ | ------------ | ------------ | ------------ | ------------ |
| app_id | int(11) unsigned | PRI | 应用ID | 100001 |
| page | varchar(128) | PRI | 页面路径 | home_page |
| page_name | varchar(16) | | 页面名称 | 首页 |
| page_desc | varchar(64) | | 页面描述 | 官网首页 |
| create_time | timestamp | | 创建时间 | 2018-07-08 14:20:00 |
| update_time | timestamp | | 更新时间 | 2018-07-09 20:10:00 |

<br/>

### tb_session_interval 会话时长区间配置表
&emsp;&emsp;按会话时长配置区间间隔信息，用于统计不同区间的会话数量。

| 字段名  | 类型  | 键 | 字段描述  | 示例
| ------------ | ------------ | ------------ | ------------ | ------------ |
| app_id | int(11) unsigned | PRI | 应用ID | 100001 |
| low | int(11) unsigned | PRI | 区间起始值 | 0 |
| show | varchar(32) | | 区间展示 | [0, 5) |

配置示例：<br/>
0   [0, 5)<br/>
5   [5, 20)<br/>
20   [20, 50)<br/>
50   [50, 200)<br/>
200   [200, 1000)<br/>
1000   [1000, )<br/>

<br/>

### tb_event_funnel 事件漏斗配置表
&emsp;&emsp;配置事件漏斗。

| 字段名  | 类型  | 键 | 字段描述  | 示例
| ------------ | ------------ | ------------ | ------------ | ------------ |
| app_id | int(11) unsigned | PRI | 应用ID | 100001 |
| funnel_id | smallint(6) unsigned | PRI | 漏斗ID | 1001 |
| activity | tinyint(1) unsigned | | 是否启用：0 未启用，1 启用 | 1 |
| funnel_name | varchar(16) | | 漏斗名称 | 注册转化 |
| funnel_desc | varchar(64) | | 漏斗描述 | 注册转化漏斗 |
| funnel_events | JSON | | 漏斗事件(JSON数组) | ["home", "register_button", "name_input", "passwd_input", "passwd_comfirm", "register_success", "login"] |
| create_time | timestamp | | 创建时间 | 2018-07-08 14:20:00 |
| update_time | timestamp | | 更新时间 | 2018-07-09 20:10:00 |



<br/>

## db_nebio_result 统计结果库
nebio相关统计结果信息存储。
### tb_user 用户统计结果表
&emsp;&emsp;存储用户统计结果数据。

| 字段名  | 类型  | 键 | 字段描述  | 示例
| ------------ | ------------ | ------------ | ------------ | ------------ |
| stat_date | date | PRI | 统计目标日期 | 2018-07-08 |
| app_id | int(11) unsigned | PRI | 应用ID | 100001 |
| channel | varchar(32) | PRI | 来源渠道 | baidu |
| user_type | tinyint(4) unsigned | PRI | 用户类型：0 未定义，1 活跃， 2 新增，3 已注册，4 访客 | 1 |
| uv | int(11) unsigned | | 用户数 | 5896 |
| pv | int(11) unsigned | | 次数 | 65839 |
| vv | int(11) unsigned | | 会话数 | 17630 |
| session_len | bigint(20) unsigned | | 总会话时长(秒) | 39583950894 |

<br/>


### tb_session 会话区间分布统计结果表
&emsp;&emsp;存储会话区间分布统计结果数据。

| 字段名  | 类型  | 键 | 字段描述  | 示例
| ------------ | ------------ | ------------ | ------------ | ------------ |
| stat_date | date | PRI | 统计目标日期 | 2018-07-08 |
| app_id | int(11) unsigned | PRI | 应用ID | 100001 |
| channel | varchar(32) | PRI | 来源渠道 | baidu |
| seg | smallint(6) unsigned | PRI | 时长区间 | 10 |
| uv | int(11) unsigned | | 用户数 | 5896 |
| pv | int(11) unsigned | | 次数 | 65839 |
| vv | int(11) unsigned | | 会话数 | 17630 |

<br/>

### tb_event 事件统计结果表
&emsp;&emsp;存储事件统计结果数据。

| 字段名  | 类型  | 键 | 字段描述  | 示例
| ------------ | ------------ | ------------ | ------------ | ------------ |
| stat_date | date | PRI | 统计目标日期 | 2018-07-08 |
| app_id | int(11) unsigned | PRI | 应用ID | 100001 |
| channel | varchar(32) | PRI | 来源渠道 | baidu |
| event_id | varchar(64) | PRI | 事件ID | home_page |
| tag | varchar(32) | PRI | 标签 | ?ADTAG=ativity |
| uv | int(11) unsigned | | 用户数 | 5896 |
| pv | int(11) unsigned | | 次数 | 65839 |
| vv | int(11) unsigned | | 会话数 | 17630 |
| iv | int(11) unsigned | | 独立IP数 | 5379 |
| event_length | bigint(20) unsigned | | 事件时长 | 39583950893 |

<br/>

### tb_page 页面统计结果表
&emsp;&emsp;存储页面统计结果数据。

| 字段名  | 类型  | 键 | 字段描述  | 示例
| ------------ | ------------ | ------------ | ------------ | ------------ |
| stat_date | date | PRI | 统计目标日期 | 2018-07-08 |
| app_id | int(11) unsigned | PRI | 应用ID | 100001 |
| channel | varchar(32) | PRI | 来源渠道 | baidu |
| page | varchar(128) | PRI | 页面 | home_page |
| uv | int(11) unsigned | | 用户数 | 5896 |
| pv | int(11) unsigned | | 次数 | 65839 |
| vv | int(11) unsigned | | 会话数 | 17630 |
| iv | int(11) unsigned | | 独立IP数 | 5379 |
| online_time | bigint(20) unsigned | | 页面停留时长(秒) | 9588323 |
| exit_vv | int(11) unsigned | | 退出数(查看该页面后退出) | 17630 |
| bounce_vv | int(11) unsigned | | 跳出数(只查看该单一页面后退出) | 657 |

<br/>


### tb_page_route 页面路径统计结果表
&emsp;&emsp;存储页面路径统计结果数据。

| 字段名  | 类型  | 键 | 字段描述  | 示例
| ------------ | ------------ | ------------ | ------------ | ------------ |
| stat_date | date | PRI | 统计目标日期 | 2018-07-08 |
| app_id | int(11) unsigned | PRI | 应用ID | 100001 |
| channel | varchar(32) | PRI | 来源渠道 | baidu |
| page | varchar(128) | PRI | 页面 | home_page |
| next_page | varchar(128) | PRI | 下一个页面 | about |
| vv | int(11) unsigned | | 用户数 | 5896 |
| pv | int(11) unsigned | | 次数 | 65839 |

<br/>

### tb_event_funnel 事件漏斗统计结果表
&emsp;&emsp;存储事件漏斗统计结果。

| 字段名  | 类型  | 键 | 字段描述  | 示例
| ------------ | ------------ | ------------ | ------------ | ------------ |
| stat_date | date | PRI | 统计目标日期 | 2018-07-08 |
| app_id | int(11) unsigned | PRI | 应用ID | 100001 |
| channel | varchar(32) | PRI | 来源渠道 | baidu |
| funnel_id | smallint(6) unsigned | PRI | 漏斗ID | 1001 |
| step | tinyint(4) unsigned | PRI | 漏斗节点序号 | 3 |
| event_id | varchar(64) | PRI | 事件ID | home_page |
| uv | int(11) unsigned | | 用户数 | 5896 |
| pv | int(11) unsigned | | 次数 | 65839 |

<br/>


