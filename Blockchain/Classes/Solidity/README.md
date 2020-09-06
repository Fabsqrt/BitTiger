# 如何实现Solidity

## Solidity的文件

每个Solidity能有多个智能合约。

### Pragma版本号

因为我们不断的在更新Solidity的版本，因此我们需要定义每个Solidity所对应的版本号。

```
progma solidity ^0.4.23;

```

上面就是一个版本号的定义，总的来说应该能够兼容到```^0.5.0```，但是除非特殊原因，不要轻易的调整版本号。否则可能会引入未知的漏洞。


### import其它文件

我们也可以把其它文件中的内容导入。

```solidity
import "filename";
```

上面的写法会把filename中的所有合约导入当下。

```solidity
import * as symbolName from "filename";
```

上面的写法会把filename中的所有合约导入到变量名symbolName内部。

在导入的时候，我们可以使用加入文件的path。

### 注释

```solidity
// This is a single-line comment.

/*
This is a
multi-line comment.
*/
```

上面是一种传统的注释方式。

```solidity
pragma solidity ^0.4.0;

/** @title Shape calculator. */
contract shapeCalculator {
    /** @dev Calculates a rectangle's surface and perimeter.
      * @param w Width of the rectangle.
      * @param h Height of the rectangle.
      * @return s The calculated surface.
      * @return p The calculated perimeter.
      */
    function rectangle(uint w, uint h) returns (uint s, uint p) {
        s = w * h;
        p = 2 * (w + h);
    }
}
```

上面使用的注释符号表示我们在给予更深入的解释。

## 智能合约的结构

智能合约内主要包含以下形式：状态变量、函数、修饰符、事件、结构体、枚举类型。我们可以把一个contract当成一个类来理解就行了。

### 状态变量

```solidity
pragma solidity ^0.4.0;

contract SimpleStorage {
    uint storedData; // State variable
    // ...
}
```

状态变量是永久存储的数据，例如storedData。

### 函数

```solidity
pragma solidity ^0.4.0;

contract SimpleAuction {
    function bid() public payable { // Function
        // ...
    }
}
```

函数是一个智能合约内可以执行的代码。

### 修饰符

```solidity
pragma solidity ^0.4.22;

contract Purchase {
    address public seller;

    modifier onlySeller() { // 修饰符
        require(
            msg.sender == seller,
            "Only seller can call this."
        );
        _;
    }

    function abort() public onlySeller { // 使用修饰符
        // ...
    }
}
```

修饰符用来对函数进行修饰。在函数执行的时候，首先要执行修饰符内的程序。因此，修饰符也是一种特殊的函数。

### 事件

```solidity
pragma solidity ^0.4.21;

contract SimpleAuction {
    event HighestBidIncreased(address bidder, uint amount); // 定义事件

    function bid() public payable {
        // ...
        emit HighestBidIncreased(msg.sender, msg.value); // 通知事件
    }
}
```

事件用来对接EVM的日志功能，简单来说就是连接外界的通知接口。

### 结构体

```solidity
pragma solidity ^0.4.0;

contract Ballot {
    struct Voter { // Struct
        uint weight;
        bool voted;
        address delegate;
        uint vote;
    }
}
```

结构体是自定义的数据结构。

### 枚举类型

```solidity
pragma solidity ^0.4.0;

contract Purchase {
    enum State { Created, Locked, Inactive } // Enum
}
```

枚举类型用来定义一些选项或状态。

## 类型

Solidity中的类型要在编译前指定，我感觉这样更加安全。我们有以下的类型：
- bool，true/false的选择
- int/uint，我们也可以定义bit的个数，例如uint8。默认是unint256。
- fixed/ufixed：fixed=fixed128*18，也就是128位是类型，18位是小数位。
- address：一个160bit的地址类型。它有特殊的成员
  - balance是余额
  - transfer用来转账
  - send是transfer的底层实现，不会抛出异常
  - call……更高级的操作，你现在也不需要掌握
- byte：字节数据，拥有length的成员
- bytes：特殊的变长的byte数组
- string：特殊的变长的字符数组
- 引用类型：通过storage定义
- array：数组；成员length表示长度；成员push在尾部添加元素
- struct：结构体
- mapping：类似于hash table的结构

## 全局变量

基本单位
- 以太单位：wei、finney、szabo、ether
- 时间单位：seconds、minutes、hours、days、weeks、years

bolck的变量
- block.后面能有很多变量

ABI函数
- abi.后接很多函数

错误处理
- assert、requrire……

数学函数
- keccak256……

地址函数
- address.后接函数

合约相关
- this...

## 智能合约

用contract开始的类


## 参考资料
- [Layout of a Solidity Source File](https://solidity.readthedocs.io/en/v0.4.23/layout-of-source-files.html)
