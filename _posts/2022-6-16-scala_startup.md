---
layout: post
title: Windows环境下Scala安装
date: 2022-06-16
tags: 计算机基础
---

1. 官网下载 `the Scala installer for Windows`[https://www.scala-lang.org/download/](https://www.scala-lang.org/download/).
2. 解压得到 cs-x86_64-pc-win32.exe, 重命名为 cs.exe (官方叫做`Coursier`) 。
3. 命令行进入cs.exe所在目录，`.\cs --help`查看帮助
4. 输入下面的命令安装scala3
```
> .\cs install scala3-compiler
> .\cs install scala3 
```
5. 配置环境变量：将`C:\Users\{用户}\AppData\Local\Coursier\data\bin`加入变量PATH。
6. 重新打开命令行，输入`scala -version`验证版本。
7. 使用Coursier安装sdt(相当于java sdk)
```
> cs setup
> sbt --script-version
```
![](/images/scala-sdt-setup-cmd.png){:height="400px" style="margin:initial"}

8. 其他方法安装sdt，打开**管理员权限**的cmd执行下述命令
```
//方法1.Chocolatey安装
> choco install sbt
//方法2.Scoop安装
> scoop install sbt
```

9. IDEA 安装SDK
- Github下载压缩包并解压 `[https://github.com/lampepfl/dotty/releases/tag/3.1.2](https://github.com/lampepfl/dotty/releases/tag/3.1.2)
- IDEA->Project Structure->Global Libraries->添加ScalaSDK->选择解压文件的lib
- 报错 ‘Not found: scala-compiler*.jar’ 不知道是IDEA版本太低，还是IDEA的Scala插件版本太低，换了2.12.16版本的scala就好了。