---
layout: post
title: AQS
date: 2020-12-30 15:12:55
tags: java
---
# 概述
本文用来描述并发包中AQS的原理,AbstractQueuedSynchronizer内部使用了双向队列实现了阻塞锁以及相关的同步器
## AQS数据结构
- 内部节点
{% asset_img 2020-12-30-16-17-58.png %}
```java
//这是AQS中的静态类
static final class Node {
 //两种模式独占和共享
 static final Node SHARED = new Node();   
 static final Node EXCLUSIVE = null;
 //5种状态常量,默认为0
 static final int CANCELLED =  1;
 static final int SIGNAL    = -1;
 static final int CONDITION = -2;
 static final int PROPAGATE = -3;
 //----属性变量----------
 Node nextWaiter; //和condition有关
 volatile int waitStatus; //当前节点状态
 volatile Node prev; //前驱
 volatile Node next; //后继
 volatile Thread thread; //持有的线程对象
 
}
```
- AQS结构
```java
private transient volatile Node head; 
private transient volatile Node tail;
private volatile int state; //改变量表示AQS本身的资源,并非是节点状态

```
- 节点状态说明
 状态0:表示节点刚加入队列  
 状态1:表示节点取消,即请求资源的线程取消动作  