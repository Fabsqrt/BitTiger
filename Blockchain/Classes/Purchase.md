# 实战智能合约：买卖物品

- 作者：冯沁原
- 网站：[www.BitTiger.io](https://www.BitTiger.io)
- 原文：[https://github.com/Fabsqrt/Blockchain](https://github.com/Fabsqrt/Blockchain)
- 邮箱：Qinyuan@BitTiger.io
- 微信：zhaxisangbo


## 需求

请实现一个买卖物品的安全协议。

## 思路

整个逻辑如下
- 卖家执行purchase的构造函数，设置一个偶数的价值
- 卖家执行购买，需要支付同样的一个价值
- 卖家发货
- 买家确认收到货，将双方的押金返回

### 代码注释

```solidity

pragma solidity ^0.4.22;

contract Purchase {
    uint public value;
    address public seller;  // 卖家地址
    address public buyer;  // 买家地址
    enum State { Created, Locked, Inactive }
    State public state;  // 当前状态

    // 构造函数
    // 确认价格msg.value是偶数
    // 具体算法是先除以2，之后再乘以2，然后比较值是否没变
    function Purchase() public payable {
        seller = msg.sender;
        value = msg.value / 2;
        require((2 * value) == msg.value, "Value has to be even.");
    }

    // 修饰符，确保_condition为true
    modifier condition(bool _condition) {
        require(_condition);
        _;
    }

    // 修饰符，确保是买家
    modifier onlyBuyer() {
        require(
            msg.sender == buyer,
            "Only buyer can call this."
        );
        _;
    }

    // 修饰符，确保是卖家
    modifier onlySeller() {
        require(
            msg.sender == seller,
            "Only seller can call this."
        );
        _;
    }

    // 修饰符，确保当前位于某个状态
    modifier inState(State _state) {
        require(
            state == _state,
            "Invalid state."
        );
        _;
    }

    event Aborted();  // 事件：停止
    event PurchaseConfirmed(); // 事件：确认购买
    event ItemReceived(); // 事件：收到物品

    /// 停止交易并回收代币
    /// 只能被卖家在合约锁定前调用
    function abort()
        public
        onlySeller
        inState(State.Created)
    {
        emit Aborted();
        state = State.Inactive;
        seller.transfer(this.balance);
    }

    /// 确认购买
    /// 交易需要包含2 * value的ether
    /// 在确认收到前，ether会被锁定
    function confirmPurchase()
        public
        inState(State.Created)
        condition(msg.value == (2 * value))
        payable
    {
        emit PurchaseConfirmed();
        buyer = msg.sender;
        state = State.Locked;
    }

    /// 买家确认收到物品，解锁ether
    function confirmReceived()
        public
        onlyBuyer
        inState(State.Locked)
    {
        emit ItemReceived();
        // 需要提前设置状态为结束
        state = State.Inactive;

        // NOTE: 这个模式有一个漏洞：卖家和买家都能阻塞转账，因此最好使用withdraw的范式

        buyer.transfer(value);
        seller.transfer(this.balance);
    }
}


```

## 总结

这是Solidity Document的一个实例，但是我觉得这个题目做的有问题。因为买家和卖家放入的是相同的押金，因此并不需要保证是偶数。

## 参考资料

- [Solidity Document](https://solidity.readthedocs.io/en/v0.4.23/introduction-to-smart-contracts.html)
