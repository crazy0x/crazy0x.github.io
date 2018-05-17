---
layout: post
title: JAVA多线程设计模式读书笔记
categories: [java]
tags: [java, multithread]
description: JAVA多线程设计模式读书笔记
---

### UML

略读

### Intro 1 Java语言的线程

- start是Thread类的方法，调用start方法，就会启动新的线程。调用start方法是，会有两个操作：1.启动新线程；2.调用run方法

- 并发与并行，并发：CPU不停切换并操作线程，同时只有一个线程的处理；并行：同时进行一个以上的处理

- synchronized方法不允许同时有一个以上的线程执行，如果有线程已经获取锁定，同一**实例**的synchronized方法并不能被其它线程所执行

- 所有的实例都有一个wait set，一执行wait方法，线程会暂时停止操作，进入wait set这个休息室，当有其它线程以notify，notifyAll，interrupt方法唤醒该线程时，线程才会退出wait set。notify和notifyAll是Object的固有方法。

### Intro 2 多线程程序的评量标准

- 安全性（必要）、存活性（必要）、复用性、性能

### Single Threaded Execution

- 何时使用？数据被多个线程访问时；SharedResource参与者状态可能发生变化时；需要确保安全性时

- STE达到下面这些条件时，有可能发生死锁：1、具有多个SharedResource参与者；2、线程锁定一个SharedResource时，还没解除锁定就提前去锁定另一个SharedResource；3、获取SharedResource参与者的顺序不固定。

- 性能低落的主要原因：1、获取锁定要花时间；2、线程冲突时必须等待。

- long与double是非原子的，如果有多线程操作，需要声明成volatile。

### Immutable

- Immutable实例的状态不会改变，因此也不需要保护。

- final类无法定义子类，声明的方法也无法被覆盖。
