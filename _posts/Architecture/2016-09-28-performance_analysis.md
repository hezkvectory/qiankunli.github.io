---

layout: post
title: 性能问题分析
category: 架构
tags: Architecture
keywords: performance

---

## 简介（待整理）

## 隔离运行环境


## 分拆项目

## 请求之间的相互影响

同一个web项目中某个url请求过于频繁，会用掉较多的tomcat线程，进而导致其他url请求速度下降。

解决办法，分拆项目，部署在不同的tomcat容器中。

## 项目之间的相互影响

### 两个项目操作共同的资源

如果一个项目对数据库表的操作过于频繁，将会影响其他项目对同一个数据库表的操作性能。

解决办法，分库分表，主从

### 两个项目相互依赖

假设a项目通过rpc形式调用b项目，发现a项目的rpc连接池异常（某段时间突然占满或占比较大），可能的原因是：

假设问题出在a项目上

1. 增加连接池的大小，继续观察
2. 连接使用后没有及时释放（现在的rpc框架一般不会有这个问题）

假设问题出来b项目上，一般主要表现在b对rpc请求的处理耗时较长，导致a无法及时释放连接。

1. 通过观察在a项目异常时间段，b项目响应时间的变化，可确认问题是否出在b项目上。

b项目出现问题的几种可能

1. 依赖服务压力过大，比如另一个rpc项目、redis和数据库等。通过观察依赖项目的response time确认。 
2. 本身代码问题。在代码的开始、结束和其它可疑位置打印时间戳，进行确认。（此类需求，一般有现成的框架）

## 进行性能分析的前提，是对系统运行情况进行全面的、准确的监控。