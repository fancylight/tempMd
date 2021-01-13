---
layout: post
title: java线程对象
date: 2021-01-13 10:16:47
tags:
---
# 概述
描述java线程(并不单单指代Thread类)的概念
## 线程状态图
{% asset_img 2021-01-13-10-17-57.png %}
1. 线程的WATTING和BLOCKING状态
前者是由`wait()`以及`Unsafe.park()`引起的,这种状态的线程是会让出锁,当被唤醒后则会进去BLOCKING状态等待唤醒
对于`sync锁`来说,当线程获得锁后,我认为会对锁对象进行描述表示存在线程持有锁对象,当线程进入WATTING状态后会修改锁状态