# 实战智能合约：公开拍卖

- 作者：冯沁原
- 网站：[www.BitTiger.io](https://www.BitTiger.io)
- 原文：[https://github.com/Fabsqrt/Blockchain](https://github.com/Fabsqrt/Blockchain)
- 邮箱：Qinyuan@BitTiger.io
- 微信：zhaxisangbo

## 需求

请实现一个拍卖协议，在该协议中，每个用户可以提交自己的出价。如果有人出价高于当前的最高价，那么我们将会退还之前的最高价的人的金额，然后将新的最高价记录在智能合约中。

## 思路

整个流程如下：
- 我们首先要记录拍卖的基本数据：谁是受益人，什么时候结束
- 我们开启拍卖，一个出价更高的人会替代之前出价最高的人
- 当出现替代时，还要退还之前出价高的人的代币
- 出于安全的考虑，退还过程将由之前用户主动发起

### 代码注解

```solidity
pragma solidity ^0.4.22;

contract SimpleAuction {
    // 拍卖的参数
    address public beneficiary; // 拍卖的受益人
    uint public auctionEnd; // 拍卖的结束时间


    address public highestBidder; // 当前的最高出价者
    uint public highestBid; // 当前的最高出价

    mapping(address => uint) pendingReturns; // 用于取回之前的出价

    bool ended; // 拍卖是否结束

    // 发生变化时的事件
    event HighestBidIncreased(address bidder, uint amount); // 出现新的最高价
    event AuctionEnded(address winner, uint amount); // 拍卖结束

    // The following is a so-called natspec comment,
    // recognizable by the three slashes.
    // It will be shown when the user is asked to
    // confirm a transaction.

    /// 创建一个拍卖，参数为拍卖时长和受益人
    function SimpleAuction(
        uint _biddingTime,
        address _beneficiary
    ) public {
        beneficiary = _beneficiary;
        auctionEnd = now + _biddingTime;
    }

    /// 使用代币来进行拍卖
    /// 当拍卖失败时，会退回代币
    function bid() public payable {
        // 不需要参数，因为都被自动处理了
        // 当一个函数要处理Ether时，需要包含payable的修饰符

        // 如果超过了截止期，交易撤回
        require(
            now <= auctionEnd,
            "Auction already ended."
        );

        // 如果出价不够，交易撤回
        require(
            msg.value > highestBid,
            "There already is a higher bid."
        );

        if (highestBid != 0) {
            // 调用highestBidder.send(highestBid)的方式是危险的
            // 因为会执行不知道的协议
            // 因此最好让用户自己取回自己的代币
            pendingReturns[highestBidder] += highestBid;
        }
        highestBidder = msg.sender;
        highestBid = msg.value;
        emit HighestBidIncreased(msg.sender, msg.value);
    }

    /// 取回被超出的拍卖前的出资
    function withdraw() public returns (bool) {
        uint amount = pendingReturns[msg.sender];
        if (amount > 0) {
            // 需要提前设置为0，因为接收者可以在这个函数结束前再次调用它
            pendingReturns[msg.sender] = 0;

            if (!msg.sender.send(amount)) {
                // 不需要throw，直接重制代币数量即可
                pendingReturns[msg.sender] = amount;
                return false;
            }
        }
        return true;
    }

    /// 结束拍卖，将金额给予受益人
    function auctionEnd() public {
        // 与其他协议交互的最好遵循以下顺序的三个步骤：
        // 1. 检查状况
        // 2. 修改状态
        // 3. 合约交互
        // 如果这三个步骤混在一起，那么攻击者可能通过多次调用这个函数来进行攻击

        // 1. 检查状况
        require(now >= auctionEnd, "Auction not yet ended.");
        require(!ended, "auctionEnd has already been called.");

        // 2. 修改状态
        ended = true;
        emit AuctionEnded(highestBidder, highestBid);

        // 3. 合约交互
        beneficiary.transfer(highestBid);
    }
}
```

## 总结

- 安全合约核心三步骤：检查状态、修改状态、合约交互
- 通过标记返还代币的方式，实现了安全的代币返还
- 通过拍卖状态，规避了多次取现的风险

## 参考资料

- [Solidity Document](https://solidity.readthedocs.io/en/v0.4.23/introduction-to-smart-contracts.html)
