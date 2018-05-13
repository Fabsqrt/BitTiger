# 实战智能合约：数据保存

- 作者：冯沁原
- 网站：[www.BitTiger.io](https://www.BitTiger.io)
- 原文：[https://github.com/Fabsqrt/Blockchain](https://github.com/Fabsqrt/Blockchain)
- 邮箱：Qinyuan@BitTiger.io
- 微信：zhaxisangbo

## 需求

请实现一个能够保存和读取一个正整数的智能合约。

## 思路

我们需要设置一个全局变量，然后对外提供两个接口
- 一个是设置
- 一个是读取

### 代码注释

```solidity

// 定义了这个文件对应的编译器的版本是0.4.23
pragma solidity ^0.4.23;

// 定义了智能合约SimpleStorage
contract SimpleStorage {
    // 定义了合约的变量storeData，它是无符号的整数，一共256位
    uint storedData;

    // 设置保存的值，是一个public，外部能够访问的函数
    function set(uint x) public {
        storedData = x;
    }

    // 读取保存的值，是一个public，外部能够访问的函数；它的返回值是uint
    function get() public constant returns (uint) {
        return storedData;
    }
}

```

以上是一段很简单的Solidity代码，任何人都能够调用它保存一个数据，并且进而得到具体的值。

## 总结

我们实现了最基本的数据读取和设置。

## 参考资料

- [Solidity Document](https://solidity.readthedocs.io/en/v0.4.23/introduction-to-smart-contracts.html)
