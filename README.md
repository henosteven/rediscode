
# redis源码分析
## redis框架

* redisServer
* redisClient
* 事件模块
	* fileevent
	* timeevent

## 基础数据结构

* sds
* ziplist
* zipmap
* intset
* dict
* skiplist
* quicklist

## 数据固化

* rdb
* aof

## 主从实现

* 配置及使用

## sentinel实现

* 配置及使用

## cluster实现

* 配置及使用

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
