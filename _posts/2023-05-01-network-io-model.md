---
layout: post
title:  "网络IO模型的个人梳理"
date:   2023-05-01 +0800
categories: network reactor
---

# 网络IO模型的个人梳理

![从网络IO模型梳理](../image/network-io-model.png)

上图梳理了个人对网络IO模型发展的认识，从最简单的单线程阻塞IO模型然后一步步完善功能发展到Reactor模型以及Proactor模型。个人理解这些模型模型并不是一蹴而就的，通过梳理可以了解到不同模型之间的差异，存在什么缺陷以及为了解决这些缺陷提出了什么方法而演进到下一个模型。

从模型的名字可以看到，这些模型主要在三个方向的组合：单线程还是多线程，同步还是异步，阻塞还是非阻塞。其中同步还是异步等到最后才来处理，在前面的模型的组合就是是否单线程和是否阻塞的组合，即：
1. 单线程同步阻塞IO(STSB Single Thread Synchronization Blocking IO)
2. 多线程同步阻塞IO(MTSB Multi Thread Synchronization Blocking IO)
3. 单线程同步非阻塞IO(STTN Single Thread Synchronization Non-blocking IO)
4. 多线程同步非阻塞IO(MTSN Multi Thread Synchronization Non-blocking IO)

在解决了以上问题之后，才提出了基于异步IO的多线程异步非阻塞IO(MTAN Mutil Thread Asynchronization Non-blocking IO)。

从整体的模型上看是这五类IO模型，到了具体的实现层面需要根据不同的场景进行优化，加入线程池和多路IO复用，以及选择基于线程的实现方案还是基于进程的实现方案衍生出了不同的实现方式

## 单线程同步阻塞IO

## 单线程同步非阻塞IO
### 缺点：CPU占用过高
### 使用多路IO复用，EPoll/Poll/Selector

## 多线程同步阻塞IO 
### 缺点：连接数过多存在问题
### 使用线程池缓解

## 多线程同步非阻塞IO —— Reactor 
### 基于线程实现的Reactor
### 基于进程实现的Reactor
### 缺点：单个连接的请求阻塞住时导致整个线程的所有连接都被阻塞
### 加入线程池

## 多线程异步非阻塞IO —— Proactor 
