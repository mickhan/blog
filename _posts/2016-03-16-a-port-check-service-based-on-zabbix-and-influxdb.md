---
layout: post
title: 基于Zabbix和Influxdb的端口健康检查服务
catalog: true
tags:
    - 监控
---

# 为何做这个东西
使用Zabbix做健康监测存在不足：频率高有压力，跨机房数据同步不稳定
但也存在好处：统一的配置管理
所以考虑附加一个健康监测服务，通过Zabbix server-proxy结构获取配置信息，使用Zabbix-agent监测健康状态，写入单独的数据库中，生成告警。
这样Zabbix只作为配置管理，不涉及数据收集和告警生成。减少了数据收集的压力，同时在很大程度上避免了因为网络波动导致的误报。

# 基本原理

## Zabbix
老牌的监控系统，各个功能已经比较完整，系统也很稳定。
Zabbix在大规模部署的情况下，常用的架构是web-server-proxy-agent。
利用server-proxy之间的配置同步机制，将配置数据复制到proxy本地的数据库中。

## Influxdb
已经有一套TICK，还不算完善，但开发相当活跃。
Influxdb作为数据存储
kapacitor作为数据整理、事件生成

## celery
python的分布式任务框架，很方便的下发任务到worker。

## 架构图
![架构图](/img/seagull.jpg "架构图")

# 一些实现细节

## Zabbix数据库结构
proxy的数据库只包含属于该proxy的数据，所以从数据库中读到的数据都是有效的。

    #获取所有host
    "select host,hostid from hosts where status=0"
    #net.tcp.port关键字，得到要监控的item
    "select key_ from items where key_ like 'net.tcp.port%' and hostid=<hostid>"
    #得到host ip
    "select ip from interface where hostid=<hostid>"
    #item获取到之后，是类似net.tcp.port[{$ip},{$port}]这样的结构，需要找到对应的宏值
    #Zabbix里面宏分布在host和template中，同时存在host-template-template这样的多叉树结构，所以需要遍历查询
    #在hostmarco中查找宏值
    "select value from hostmacro where hostid=<hostid> and macro=<key_>"
    "没有找到的话，在hosts_templates中搜索它的模板
    "SELECT templateid FROM hosts_templates where hostid=<hostid>"
    "然后再查找模板里面有没有对应的宏值"
    "select value from hostmacro where hostid=<templateid> and macro=<key_>"

最后得到了hostip bindip port，这就是我们需要监控的目标。

## Zabbix_get获取数据
使用一个开源的[Python port of Zabbix_get command](https://gist.github.com/quiver/3883369)

## kapacitor告警规则

### last n value
[How to check last n continue value · Issue #281 · influxdata/kapacitor](https://github.com/influxdata/kapacitor/issues/281)

### nodata
<a href="https://github.com/influxdata/kapacitor/issues/137">Make creating an alert based on event count easy · Issue #137 · influxdata/kapacitor</a>

### edge trigger
好像并没有这个功能，我是在发送告警的脚本中加了标志位。

### recovery
异常恢复时会自动生成recovery消息，Level为"OK"

