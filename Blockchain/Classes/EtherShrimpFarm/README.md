# 如何实现Ether Shrimp Farm

Disclaimer: None of this is financial advice

- 作者：冯沁原
- 网站：[www.BitTiger.io](https://www.BitTiger.io)
- 原文：[https://github.com/Fabsqrt/](https://github.com/Fabsqrt/)
- 邮箱：Qinyuan@BitTiger.io
- 微信：zhaxisangbo

## 视频讲解

[![](http://img.youtube.com/vi/CsLYK7mz2P0/0.jpg)](http://www.youtube.com/watch?v=CsLYK7mz2P0 "")


## Ether Shrimp Farm长什么样

每个新玩家进来，都能获得300个免费的虾，每只虾每秒钟产下1个虾籽，这些虾籽会累积起来，最多持续累积一天。

对于累计的虾籽，你可以选择卖掉换成以太币，或者按照86400的比例转换成一只虾。

你买入和卖出虾的价格有什么决定呢？

作者怎么挣钱呢？5%的交易费会进入作者的口袋。

如果你推荐了人加入，那么他每次把虾籽孵化为虾的时候，你会收到20%的虾籽。

这个游戏的特点是，你什么都不需要支付，就可以得到免费的300只虾。于是极大的增加的大家的参与度。当然你还是需要支付gas费用的。于是EOS就特别适合。


## 思考

```solidity

pragma solidity ^0.4.18; // solhint-disable-line



contract ShrimpFarmer{
    //uint256 EGGS_PER_SHRIMP_PER_SECOND=1; // QY，猜想作者开始想按照秒数来计算下籽的逻辑，但是后来换成了按照1天来计算的方式
    uint256 public EGGS_TO_HATCH_1SHRIMP=86400;// 通过一整天的时间（以秒为单位）计算孵化虾籽的情况
    uint256 public STARTING_SHRIMP=300; // 开始给一个人的虾的数量
    uint256 PSN=10000;
    uint256 PSNH=5000;
    bool public initialized=false; // 是否完成初始化
    address public ceoAddress; // CEO的地址
    mapping (address => uint256) public hatcheryShrimp; // 正在下籽的虾的数量
    mapping (address => uint256) public claimedEggs; // 保存用户购买的虾籽和推荐得到的虾籽总数，但是不包含池塘中的虾产下来的虾籽
    mapping (address => uint256) public lastHatch; // 最后一次操作的时间
    mapping (address => address) public referrals; // 推荐人
    uint256 public marketEggs; // 市场虾籽数的评估指标
    function ShrimpFarmer() public{
        ceoAddress=msg.sender; // 设置创建者为CEO
    }
    function hatchEggs(address ref) public{ // 把虾籽孵化为虾
        require(initialized); // 需要完成平台初始化的过程
        if(referrals[msg.sender]==0 && referrals[msg.sender]!=msg.sender){ // 当自己没有推荐人，并且自己保存的推荐人不是自己，进行更新；QY，代码有可能写错了，应该是ref!=msg.sender
            referrals[msg.sender]=ref; // 设置推荐人
        }
        uint256 eggsUsed=getMyEggs(); // 得到自己的虾籽的数量
        uint256 newShrimp=SafeMath.div(eggsUsed,EGGS_TO_HATCH_1SHRIMP); // 一天的秒数作为除数，计算孵化出来的虾的数量
        hatcheryShrimp[msg.sender]=SafeMath.add(hatcheryShrimp[msg.sender],newShrimp); //
        claimedEggs[msg.sender]=0;
        lastHatch[msg.sender]=now; // 记录最后一次孵化虾籽的时间为当前时间

        //send referral eggs
        claimedEggs[referrals[msg.sender]]=SafeMath.add(claimedEggs[referrals[msg.sender]],SafeMath.div(eggsUsed,5)); // 推荐者获得使用的虾籽的20%

        //boost market to nerf shrimp hoarding
        marketEggs=SafeMath.add(marketEggs,SafeMath.div(eggsUsed,10)); // 增加市场上虾籽的数量，增加消耗掉的虾籽数量除以10
    }
    function sellEggs() public{
        require(initialized); // 需要完成平台初始化过程
        uint256 hasEggs=getMyEggs(); // 得到虾籽的数量
        uint256 eggValue=calculateEggSell(hasEggs); // 计算虾籽的价值
        uint256 fee=devFee(eggValue); // 计算费用
        claimedEggs[msg.sender]=0; // 清空
        lastHatch[msg.sender]=now; // 更新最后的时间
        marketEggs=SafeMath.add(marketEggs,hasEggs); // 更新市场上虾的数量，增加卖掉的虾籽总数
        ceoAddress.transfer(fee); // 把费用给CEO
        msg.sender.transfer(SafeMath.sub(eggValue,fee)); // 给出售者收益
    }
    function buyEggs() public payable{
        require(initialized); // 需要完成平台初始化过程
        uint256 eggsBought=calculateEggBuy(msg.value,SafeMath.sub(this.balance,msg.value)); // 基于出价计算购买的数量
        eggsBought=SafeMath.sub(eggsBought,devFee(eggsBought)); // 减少数量
        ceoAddress.transfer(devFee(msg.value)); // ceo给自己转账
        claimedEggs[msg.sender]=SafeMath.add(claimedEggs[msg.sender],eggsBought); // 给自己添加虾籽
    }
    //magic trade balancing algorithm
    function calculateTrade(uint256 rt,uint256 rs, uint256 bs) public view returns(uint256){ // 基于卖的虾籽的数量，市场上虾籽的数量，自己的账户余额
        //(PSN*bs)/(PSNH+((PSN*rs+PSNH*rt)/rt)); // 计算公式 10000, 5000
        // bs / ( 1 + rs/rt )
        return SafeMath.div(SafeMath.mul(PSN,bs),SafeMath.add(PSNH,SafeMath.div(SafeMath.add(SafeMath.mul(PSN,rs),SafeMath.mul(PSNH,rt)),rt)));
    }
    function calculateEggSell(uint256 eggs) public view returns(uint256){ // 计算对应虾籽数量的价值
        return calculateTrade(eggs,marketEggs,this.balance);
    }
    function calculateEggBuy(uint256 eth,uint256 contractBalance) public view returns(uint256){
        return calculateTrade(eth,contractBalance,marketEggs); // 三个参数
    }
    function calculateEggBuySimple(uint256 eth) public view returns(uint256){
        return calculateEggBuy(eth,this.balance); // 传入ETH，添加自己的balance为参数
    }
    function devFee(uint256 amount) public view returns(uint256){
        return SafeMath.div(SafeMath.mul(amount,4),100); // 乘以4，除以100，也就是4%的费用
    }
    function seedMarket(uint256 eggs) public payable{ // QY，感觉payable可以不用
        require(marketEggs==0); // 市场上没有任何虾籽，初始化的过程
        initialized=true; // 已经开启了初始化过程
        marketEggs=eggs; // 设置虾籽的数量的初始值
    }
    function getFreeShrimp() public{
        require(initialized); // 需要已经完成了初始化
        require(hatcheryShrimp[msg.sender]==0); // 需要调用者的虾的数量为空
        lastHatch[msg.sender]=now; // 设置最后操作时间为当下
        hatcheryShrimp[msg.sender]=STARTING_SHRIMP; // 设置初始赠送的虾300只
    }
    function getBalance() public view returns(uint256){ // 得到账上余额
        return this.balance;
    }
    function getMyShrimp() public view returns(uint256){ // 得到调用者虾的数量
        return hatcheryShrimp[msg.sender];
    }
    function getMyEggs() public view returns(uint256){ // 得到调用者虾籽的数量，由两部分组成，自己可以claim的虾的数量和距离上次新生产出的虾籽的数量
        return SafeMath.add(claimedEggs[msg.sender],getEggsSinceLastHatch(msg.sender));
    }
    function getEggsSinceLastHatch(address adr) public view returns(uint256){
        uint256 secondsPassed=min(EGGS_TO_HATCH_1SHRIMP,SafeMath.sub(now,lastHatch[adr])); // 记录经过了多久的虾的统计时间
        return SafeMath.mul(secondsPassed,hatcheryShrimp[adr]); // 每一秒都能产生一个新的虾籽
    }
    function min(uint256 a, uint256 b) private pure returns (uint256) { // 比较两个数大小
        return a < b ? a : b;
    }
}

library SafeMath {

  /**
  * @dev Multiplies two numbers, throws on overflow.
  */
  function mul(uint256 a, uint256 b) internal pure returns (uint256) { // 安全乘法
    if (a == 0) {
      return 0;
    }
    uint256 c = a * b;
    assert(c / a == b);
    return c;
  }

  /**
  * @dev Integer division of two numbers, truncating the quotient.
  */
  function div(uint256 a, uint256 b) internal pure returns (uint256) { // 安全除法
    // assert(b > 0); // Solidity automatically throws when dividing by 0
    uint256 c = a / b;
    // assert(a == b * c + a % b); // There is no case in which this doesn't hold
    return c;
  }

  /**
  * @dev Substracts two numbers, throws on overflow (i.e. if subtrahend is greater than minuend).
  */
  function sub(uint256 a, uint256 b) internal pure returns (uint256) { // 安全减法
    assert(b <= a);
    return a - b;
  }

  /**
  * @dev Adds two numbers, throws on overflow.
  */
  function add(uint256 a, uint256 b) internal pure returns (uint256) { // 安全加法
    uint256 c = a + b;
    assert(c >= a);
    return c;
  }
}

```

## 关于赚钱的机智地思考

现在以太坊上的开发，都是以投机为主。现在的主流是基于博弈论的原理设计，基于一个简单的投资原则，让玩家进行博弈。开发者收取5%左右的交易费。


## 参考资料

- [http://ethershrimpfarm.net/](http://ethershrimpfarm.net/)
