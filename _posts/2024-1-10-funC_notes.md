---
layout: post
title: FunC语法学习笔记
date: 2023-1-10
tags: 区块链
---
## FunC语言
### 简述
- 类C语言，区块链TON链专用

### 变量作用域
- 声明全局变量(global)时，类型可以省略，会通过使用这个变量的地方推断这个变量的类型。（个人觉得代码难读，容易出错）
- 如果在局部声明一个和全局变量相同的变量名，会认为它就是那个全局变量（即声明无效，直接指向同名的全局变量）。如果声明的类型与全局变量的类型不同，代码会无法通过编译。

### 函数修饰符
- impure：标记这个方法会修改storage变量。如果一个方法不标记impure，也没有被别的方法引用，那它会被编译忽视。
- inline：标记这个方法，任意调用这个方法的地方会在编译时内联代码。省略了执行方法是查方法字典的开销，gas_fee会更便宜。
- inline_ref：与inline相似。内联到方法的引用(jump to the reference)
- 所以读方法和工具方法尽量声明pure和view，可以省钱。
- forall: 多态修饰符。作用有点像java的泛型。(doc_link)[https://docs.ton.org/develop/func/functions#polymorphism-with-forall]

### 变量的坑
- 声明了`int x`， 表示x的相反数时，`- x`是正确的；`-x`是错误的，因为它符合变量的命名规则，被视为一个变量。

### 事件

### 继承

### 抽象和接口

### 异常

## 智能合约开发教程
### 1. 创建钱包和连接tonCenter客户端
0. [doc_link](https://ton-community.github.io/tutorials/01-wallet/)
1. 安装基础包
- `npm install ts-node`
- `npm install @ton/ton @ton/crypto @ton/core`
2. 在包含ts-node包的环境运行step7.ts(如果没有安装ts-node，会临时下载包并在运行结束后删除)
- `npx ts-node step7.ts`
3. 安装tonCenter客户端终端包
- `npm install @orbs-network/ton-access`
### 2. 编写智能合约
0. [doc_link](https://ton-community.github.io/tutorials/02-contract/)
1. `node -v`检查nodejs版本（官方推荐v18以上）
2.  创建项目
- `npm create ton@latest`
- 包含文件夹`contracts`,`wrappers`,`tests`,`scripts`, 并自行创建文件夹`build`
- **contracts**: 放fc源码
- **wrappers**: 使用js执行fc代码的包装类（脚本）
- **tests**: 放js测试代码。通过wrappers里的js包装类来测试编写的fc代码。
- **scripts**: 放js脚本。比如部署fc合约的脚本。
- **build**: 放fc编译后的bytecode文件。类似java项目的`./target/classes`
3. 编译fc文件
- `npx func-js contracts/counter.fc --boc build/counter.cell`
4. 编写js包装类(wrappers)作为inferface调用bytecode
5. 部署合约(bytecode)
- `npm install @orbs-network/ton-access` 安装tonCenter终端
- `npx blueprint run` 使用blueprint工具部署合约，依次选`{network}`,`Create a ton:// deep link`
### 3. 测试智能合约（部署测试网、主网省略，仅介绍单元测试）
0. [doc_link](https://ton-community.github.io/tutorials/04-testing/)
1. 安装ts测试包和ton沙盒，检查`tsconfig.json`和`jest.config.js`文件是否创建
- `npm install typescript jest @types/jest ts-jest`
- `npm install @ton/core @ton/sandbox @ton/test-utils`
2. 运行测试文件
- `npx jest step2`或`npx jest .\step2.spec.ts`（建议在根目录运行，运行时目录会影响到测试代码中的文件相对路径）
3. 最后部署到主网测试吧


## QA
1. 怎么理解global变量和storage的区别
> 在TON链中，Global变量和Storage是两种不同类型的存储方式，它们各自具有特定的特性和用途。
Global变量是一种全局变量，它可以在合约的所有函数和方法中被访问。Global变量通常用于存储合约的状态信息，如合约的配置参数、全局计数器等。在合约部署时，Global变量的初始值可以被设置，并且在合约的生命周期内保持不变，除非在合约代码中有相应的逻辑对其进行修改。
Storage则是一种持久化存储方式，用于存储合约的长期状态数据。与Global变量不同的是，Storage中的数据可以被合约的各个方法和函数动态修改，并且这些修改会被永久保存在区块链上。因此，通常情况下，存储在Storage中的数据会更加重要和敏感，比如用户的资产信息、交易记录等。
总的来说，Global变量用于存储合约的全局状态信息，而Storage则用于持久化存储合约的长期状态数据。合理使用这两种存储方式可以帮助合约实现更加灵活和高效的逻辑和功能。
可以配合使用，从storage读到数据赋值给global变量，然后就能使用global变量。或者写storage赋值给global变量，再写一个save方法刷到storage（类似cache-flush的设计）