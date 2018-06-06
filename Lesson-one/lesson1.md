# **lesson1**
========
## 预先学习
服务器配置
secureRCT 安装
rpc调用
练习leaf
学习html5

========
## 熟悉leaf 

#### leaf/chanrpc 提供了一套基于 channel 的 RPC 机制，用于游戏服务器模块间通讯
#### leaf/db 数据库相关，目前支持 MongoDB 
#### leaf/gate 网关模块，负责游戏客户端的接入
#### leaf/go 用于创建能够被 Leaf 管理的 goroutine 
#### leaf/log 日志相关
#### leaf/network 网络相关，使用 TCP 和 WebSocket 协议，可自定义消息格式，默认 Leaf 提供了基于 protobuf 和 JSON 的消息格式
#### leaf/recordfile 用于管理游戏数据
#### leaf/timer 定时器相关
#### leaf/util 辅助库

## 作业
leaf rpc 通信 
a -rpc- b  -rpc- c
a模块通过b模块 获得C模块的数据
是否能够成功
