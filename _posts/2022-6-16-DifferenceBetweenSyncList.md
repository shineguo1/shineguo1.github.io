---
layout: post
title: 线程安全List选型
date: 2022-06-16
tags: 计算机-1分钟知识
---

|       |Vector                     |SynchronizedList                   |CopyOnWriteArrayList|
|-      |-                          |-                                  |-                   |
|原理   |用synchronized 修饰读写方法| 读写方法用synchronized代码块加锁   |写操作的时候复制数组|
|互斥   |-                          |-                                   |读读操作和读写操作不互斥，写写互斥|
|写性能 |803ms/千万次               |5735ms/千万次                       |4274ms/**十万**次  |
|读性能 |433ms/千万次               |402ms/千万次                        |97ms/千万次        |
|场景   |有写要求                   |有写要求                            |读远远大于写(写操作非常耗时)|

---
- org.apache.commons.collections.list.SynchronizedList
- java.util.Collections.SynchronizedList
