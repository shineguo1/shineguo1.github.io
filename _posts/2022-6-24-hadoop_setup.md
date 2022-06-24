---
layout: post
title: windows下hadoop环境搭建
date: 2022-06-24
tags: 计算机基础
---
### 〇、资源
[hadoop官网3.2.3版本下载](https://archive.apache.org/dist/hadoop/common/hadoop-3.2.3/) <br/>
[winutils-master 3.2.x gihub下载](https://github.com/cdarlint/winutils)

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