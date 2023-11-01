---
layout: post
title: 大数据-分布式技术概述
date: 2022-07-21
tags: 计算机基础
---
### 一、hadoop
1. 特点：
- 高可靠性：多副本（不同服务器）。某个计算元素/存储故障，数据不会丢失。
- 高扩展性：集群间分配任务数据，可以横向扩展服务器。
- 高效性：在MapReduce思想下，并行工作。
- 高容错性：失败的任务自动重新分配。
2. HDFS: 分布式数据存储
- NameNode：存储文件元数据，如文件名、目录结构、文件属性，以及每个文件的块列表、块所在dataNode
- DataNode：具体存储数据，即文件块数据、块数据校验和
- 2NN(secondary name node)：每个一段时间对NameNode元数据备份
3. MapReduce：分布式计算（版本1.X计算+资源调度，2.X仅计算）
- Map（映射）阶段把数据分发到多台机器，并行处理输入数据。
- Reduce（汇总）阶段对Map结果进行汇总。

4. Yarn：资源调度（版本2.X新增，负责资源调度）
- ResourceManager (RM):整个集群资源(内存、CPU等)的老大
- NodeManager (NM) :单个节点服务器资源老大
- ApplicationMaster (AM)∶单个任务运行的老大
- Container:容器，相当一台独立的服务器，里面封装了任务运行所需要的资源，如内存、CPu、磁盘、网络等。

### 一、官网安装

1. 下载`hadoop-3.2.3.tar.gz`，url可改成想要的版本
2. 下载`winutils-master`, 选择对应的版本（最后一位数字可以不一样）
3. 解压`hadoop-3.2.3.tar.gz`
4. 将`winutils-master`内的文件拷贝并替换到`./hadoop-3.2.3/bin`
5. 将`winutils-master`内的`hadoop.dll`文件拷贝到`C:\Windows\System32`
6. 设置环境变量
- 新建HADOOP_HOME，值为解压后根目录，如`D:\Program_Files\hadoop\hadoop-3.2.3`
- 添加到path路径，`%HADOOP_HOME%\bin`

6. 运行cmd，如下表示成功：
```
C:\Users\xxxxx>hadoop -version
java version "11.0.15.1" 2022-04-22 LTS
Java(TM) SE Runtime Environment 18.9 (build 11.0.15.1+2-LTS-10)
Java HotSpot(TM) 64-Bit Server VM 18.9 (build 11.0.15.1+2-LTS-10, mixed mode)
```


### 二、常见问题
1. 出现问题1：
```
'hadoop' 不是内部或外部命令，也不是可运行的程序或批处理文件。
```
解决：这是由`winutils-master`与`hadoop-3.2.3.tar.gz`版本不一致引起的
2. 出现问题2：
```
Error : JAVA_HOME is incorrectly set.
P1ease _update D:\xxxxxx\etc\hadoop\hadoop-env.cmd
’-Xmx512m’不是内部或外部命令，也不是可运行的程序或批处理文件。
```
解决：问题出在hadoop要求JAVA_HOME路径不能包含空格。处理步骤如下：
- 拷贝jdk到一个不包含空格的路径
- 根据提示编辑`hadoop-env.cmd`文件, 修改`JAVA_HOME`的值为刚刚的新路径
3. 出现问题3：
```
本地安装成功，IDEA依旧报错：HADOOP_HOME and hadoop.home.dir are unset
```
解决：重启IDEA