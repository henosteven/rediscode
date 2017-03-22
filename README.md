
# redis源码分析
## redis框架

* redisServer
* redisClient
* redisDb
* redisObject
* 命令实现
* 定时任务
* 命令实现
* 事件模块
	* fileevent
	* timeevent

## 基础数据结构

* [sds](./datastruct/sds.md)
* [ziplist](./datastruct/ziplist.md)
* zipmap
* intset
* dict
* skiplist
* quicklist

## redis命令分析

* list
* hashtable
* string
* set
* sortset

## 数据固化

* rdb
	* 相关配置
	* 实现
* aof
	* 相关配置
	* 实现
	
## 主从实现

* 配置及使用
* 实现原理

## sentinel实现

* 配置及使用
* 实现原理

## cluster实现

* 配置及使用
* 实现原理

## redis的一些额外封装

* pub|sub
* watch | multi exec
* rio
* bio
* latency
* slowlog
* blpop

## redis的一些命令技巧

* rpoplpush
