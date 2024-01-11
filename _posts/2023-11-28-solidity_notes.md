---
layout: post
title: solidity语法学习笔记
date: 2023-11-28
tags: 区块链
---
### 简述
- solidity是面向对象语言，有c、java、js基础的很容易看懂，仅列举一些需要注意的点

### 变量作用域
- 引用类型包含所有struct和array，隐性的有bytes、bytes32、string
- 分为状态变量、局部变量、全局变量。状态变量等同于类的成员变量，局部变量等同于方法内或代码段内的变量，全局变量单独解释。
- 全局变量：它是solidity的预留关键字，不是由合约自己定义的，有点像系统变量、系统函数。它在任意地方都能直接使用。如`msg.data`,`msg.sender`,`block.number`,`block.timestamp`,`gasleft()`。

### 函数修饰符
- pure：不可读状态变量，不可写状态变量。不用付gas。
- view：可读状态变量，不可写状态变量。不用付gas。
- 缺省：可读状态变量，可写状态变量。需要付gas。
- 所以读方法和工具方法尽量声明pure和view，可以省钱。

### 变量的坑
- 常用的数值类型uint是无符号数，当计算过程中变成负数，会报错！！所以像`if(num1 - num2 > 0)`这样的语句不适用于uint数值类型，违反其他语言养成的习惯(c、java、js一般都用有符号数)。

### 事件
- indexed topics：最多只有4个长度，其中事件的签名占一位，所以最多只能存放3个字段。
- data：大小不限。

### 继承
- 祖父类：Human.sol, 父类：Father.sol，GrandFather.sol， 子类：Son.sol
```js
// Human.sol
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.4;

abstract contract Human {
    //event
    //===============
    event Log(string msg);

        //method
    //================
    function foo() public virtual returns(string memory){
        emit Log("Human");
        return "Human";
    }

    // function abs_method() public virtual;
}

//===============================================================
// Grandfather.sol
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.4;

import "./Human.sol";

contract Grandfather is Human{

    //method
    //================
    function foo() public override virtual returns(string memory){
        emit Log("Grandfather");
        //问题1：如果删掉这里的super.foo()会怎样？
        //问题2：如果保留这里的super.foo()会怎样？
        super.foo();
        return "Grandfather";
    }

    function doGrandfather() public virtual returns(string memory){
        emit Log("Grandfather");
        return "Grandfather";
    }
}

//===============================================================
// Father.sol
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.4;

import "./Human.sol";

contract Father is Human {

    //method
    //================
    function foo() public override virtual returns(string memory){
        emit Log("father");
        super.foo();
        return "father";
    }

    function doFather() public virtual returns(string memory){
        emit Log("father");
        return "father";
    }
}

//===============================================================
// Son.sol
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.4;

import "./Father.sol";
import "./Grandfather.sol";

contract Son is Grandfather,Father {

    //method
    //================
    function foo() public override(Grandfather,Father) virtual returns(string memory){
        emit Log("son");
        //可以写Grandfather.foo(),指名调用哪个父类的实现。使用super如下
        super.foo(); //直接super调用的是最近的父类函数
                    //在多重继承中，Son在Father类中的super调用到了Son的第二近的父类GrandFather
                    //Son没有其他父类了，Grandfather的super方法才调用到自己的父类Human
        return "son";
    }

    function doSon() public virtual returns(string memory){
        emit Log("son");
        return "son";
    }
}

```
- 在多重继承下（嵌套super.foo()），如果调用Son.foo()会怎么样？
- 如果GrandFather中删除super.foo()， 那么Logs显示的调用顺序是Son->Father->Grandfather, 而不是Son->Father->Human. 因为第二次在Father代码中执行super.foo(), 调用的还是Son的父类，第二近的父类是Grandfather。
-  如果GrandFather中保留super.foo()， 那么Logs显示的调用顺序是Son->Father->Grandfather->Human. 因为第三次在Grandfather中调用super.foo()，Son的父类遍历完了于是才访问到父类的父类Human。
- super的调用顺序是图遍历，从近到远依次调用图中的节点（把父类和祖父类绘成图，所以不会出现因为2个父类的祖父类是同一个类，而调用了2次祖父类）
> 图如下，括号内表示节点的优先级
> Son(0)->Father(1)
> Son(0)->Grandfather(2)
> Father(1)->Human(3)
> Grandfather(2)->Human(3)

### 抽象和接口
- 抽象方法不需要abstract关键字声明，只要不实现方法体即可。包含抽象方法的类必须是抽象类。
- 接口只能有定义（event、error和抽象方法），不能有实现。

### 异常
- 使用`error MyException(parameter); revert MyException`代替`require(expr1, "this is errorMsg")`。因为error更省gas。

### 接受ETH的回调函数
- 任意方法，用payable修饰，其他合约就可以在调用这个方法时添加花括号`myContract.payable_method{value: eth_amount}`来传递eth，这个合约就可以在这个方法里执行后置代码逻辑。
- `receive() payable external`, 是接受tansfer、call{value: amount}("")、send方法传递eth时的默认回调方法。
- `fallback() payable external`, 是接受tansfer、call{value: amount}("")、send方法传递eth，并且receive方法不存在时的回调方法。根据单一职责原则，fallback()应该处理回退逻辑，接受eth后置代码逻辑推荐使用receive方法实现。
- `recive()`和`fallback`是约定的函数，如果这个函数都没有实现，则合约不能接受tansfer、call{value: amount}("")、send方法传递eth


### 发送ETH
- call没有gas限制，最为灵活，是最提倡的方法；
- transfer有2300 gas限制，但是发送失败会自动revert交易，是次优选择；
- send有2300 gas限制，而且发送失败不会自动revert交易，几乎没有人用它。

### delegateCall
1. A -> B -call-> C
- 此时执行C合约代码时，语境是B->C，即msg.sender = B, msg.value = B.call时传的value. 方法中读写的变量值是C合约的状态变量。
2. A -> B(Proxy Contract, 职责是存储数据) -delegateCall-> C(Logic Contract, 部署逻辑代码)
- 此时执行C合约代码时，语境是A->B，即msg.sender = A, msg.value = A.call时传value。方法中读写的变量值是B合约的状态变量，不会影响到C合约的状态变量。


### 创建合约
- create: 新地址 = hash(创建者地址, nonce), 代码是`new Contract{value: amount}()`, 创建的合约地址是随机的。
- create2：新地址 = hash("0xFF",创建者地址, salt, bytecode)， 代码是`new Contract{salt: _salt, value: _value}()`, 创建的合约地址是确定的，因为用可控的salt代替了不可控的nonce。所以使用这个方法创建合约，可以使得多条链上合约地址相同。

`