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


## db_nebio_result 统计结果库
nebio相关统计结果信息存储。
### tb_event_stat 事件统计结果表
&emsp;&emsp;存储事件统计结果数据。

| 字段名  | 类型  | 键 | 字段描述  | 示例
| ------------ | ------------ | ------------ | ------------ | ------------ |
| stat_date | date | | 统计目标日期 | 2018-07-08 |
| app_id | int(11) unsigned | PRI | 应用ID | 100001 |
| event_id | varchar(64) | PRI | 事件ID | home_page |
| channel | varchar(32) | PRI | 来源渠道 | baidu |
| tag | varchar(32) | PRI | 标签 | ?ADTAG=ativite |
| pv | int(11) unsigned | | 次数 | 65839 |
| uv | int(11) unsigned | | 用户数 | 5896 |
| vv | int(11) unsigned | | 会话数 | 17630 |
| iv | int(11) unsigned | | 独立IP数 | 5379 |
| event_length | bigint(20) unsigned | | 事件时长 | 39583950893 |


