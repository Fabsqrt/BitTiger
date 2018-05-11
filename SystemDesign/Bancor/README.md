# 如何实现基于以太坊的交易所Bancor

- 作者：冯沁原
- 网站：[www.BitTiger.io](https://www.BitTiger.io)
- 原文：[github.com/Fabsqrt/BitTigerLab](https://github.com/Fabsqrt/BitTigerLab)
- 邮箱：Qinyuan@BitTiger.io


## Bancor

支撑Bancor的核心是智能代币，也就是通过智能协议的连接器把多个代币连接起来，从而能够进行代币间有效的转换。

Bancor Formula是具体的转换算法，它维持了代币间的一个常数的关系。

这样做去除了交易的第二方，任何人都能在任何时刻交易自己手中的代币。

BNT就是Bancor Network Token，是串联Bancor网络中所有代币的代币，它以ETH为基准。

## 源码分析

```solidity

pragma solidity ^0.4.18;

/*
    工具和修饰器
*/
contract Utils {
    /**
        构建函数
    */
    function Utils() public {
    }

    // 验证数量是否大于0
    modifier greaterThanZero(uint256 _amount) {
        require(_amount > 0);
        _;
    }

    // 验证地址是否为空
    modifier validAddress(address _address) {
        require(_address != address(0));
        _;
    }

    // 验证地址与当前的地址不同
    modifier notThis(address _address) {
        require(_address != address(this));
        _;
    }

    // 防止溢出的运算函数

    /**
        @dev returns the sum of _x and _y, asserts if the calculation overflows

        @param _x   value 1
        @param _y   value 2

        @return sum
    */
    function safeAdd(uint256 _x, uint256 _y) internal pure returns (uint256) {
        uint256 z = _x + _y;
        assert(z >= _x);
        return z;
    }

    /**
        @dev returns the difference of _x minus _y, asserts if the subtraction results in a negative number

        @param _x   minuend
        @param _y   subtrahend

        @return difference
    */
    function safeSub(uint256 _x, uint256 _y) internal pure returns (uint256) {
        assert(_x >= _y);
        return _x - _y;
    }

    /**
        @dev returns the product of multiplying _x by _y, asserts if the calculation overflows

        @param _x   factor 1
        @param _y   factor 2

        @return product
    */
    function safeMul(uint256 _x, uint256 _y) internal pure returns (uint256) {
        uint256 z = _x * _y;
        assert(_x == 0 || z / _x == _y);
        return z;
    }
}

/*
    协议：拥有
*/
contract IOwned {
    // this function isn't abstract since the compiler emits automatically generated getter functions as external
    function owner() public view returns (address) {}

    function transferOwnership(address _newOwner) public;
    function acceptOwnership() public;
}

/*
    协议：提供所有权协约的支持
*/
contract Owned is IOwned {
    address public owner;
    address public newOwner;

    event OwnerUpdate(address indexed _prevOwner, address indexed _newOwner);

    /**
        @dev 构建函数
    */
    function Owned() public {
        owner = msg.sender;
    }

    // 只允许主人
    modifier ownerOnly {
        assert(msg.sender == owner);
        _;
    }

    /**
        @dev 支持更换主人
        新主人还需要在后续接受转换
        只能被当前主人调用

        @param _newOwner    协议的新主人
    */
    function transferOwnership(address _newOwner) public ownerOnly {
        require(_newOwner != owner);
        newOwner = _newOwner;
    }

    /**
        @dev 协议的新主人接受成为新主人
    */
    function acceptOwnership() public {
        require(msg.sender == newOwner);
        OwnerUpdate(owner, newOwner);
        owner = newOwner;
        newOwner = address(0);
    }
}

/*
    提供协议管理的支持
*/
contract Managed {
    address public manager;
    address public newManager;

    event ManagerUpdate(address indexed _prevManager, address indexed _newManager);

    /**
        @dev 构建函数
    */
    function Managed() public {
        manager = msg.sender;
    }

    // 只允许经理执行
    modifier managerOnly {
        assert(msg.sender == manager);
        _;
    }

    /**
        @dev 允许更改管理员
        新的管理员需要进行确认
        只能被当前管理员调用

        @param _newManager    新的协议管理员
    */
    function transferManagement(address _newManager) public managerOnly {
        require(_newManager != manager);
        newManager = _newManager;
    }

    /**
        @dev 接受成为洗呢管理员
    */
    function acceptManagement() public {
        require(msg.sender == newManager);
        ManagerUpdate(manager, newManager);
        manager = newManager;
        newManager = address(0); // 避免二次确认
    }
}

/*
    ERC20 标准代币接口
*/
contract IERC20Token {
    // these functions aren't abstract since the compiler emits automatically generated getter functions as external
    function name() public view returns (string) {}
    function symbol() public view returns (string) {}
    function decimals() public view returns (uint8) {}
    function totalSupply() public view returns (uint256) {}
    function balanceOf(address _owner) public view returns (uint256) { _owner; }
    function allowance(address _owner, address _spender) public view returns (uint256) { _owner; _spender; }

    function transfer(address _to, uint256 _value) public returns (bool success);
    function transferFrom(address _from, address _to, uint256 _value) public returns (bool success);
    function approve(address _spender, uint256 _value) public returns (bool success);
}

/*
    智能代币接口
*/
contract ISmartToken is IOwned, IERC20Token {
    function disableTransfers(bool _disable) public;
    function issue(address _to, uint256 _amount) public;
    function destroy(address _from, uint256 _amount) public;
}

/*
    代币拥有者接口
*/
contract ITokenHolder is IOwned {
    function withdrawTokens(IERC20Token _token, address _to, uint256 _amount) public;
}

/*
    我们把每个协议当成一个代币拥有者，因为一个协议不能拒绝接收代币

    代币拥有者的协议的目的是提供一个安全机制，允许主人把误发的代币返回给发送者
*/
contract TokenHolder is ITokenHolder, Owned, Utils {
    /**
        @dev 构建函数
    */
    function TokenHolder() public {
    }

    /**
        @dev 提取协议的代币，并且发给一个账号
        只能被主人调用

        @param _token   ERC20 token contract address
        @param _to      account to receive the new amount
        @param _amount  amount to withdraw
    */
    function withdrawTokens(IERC20Token _token, address _to, uint256 _amount)
        public
        ownerOnly
        validAddress(_token)
        validAddress(_to)
        notThis(_to)
    {
        assert(_token.transfer(_to, _amount));
    }
}

/*
    智能代币管理器是一个可以升级的模块，从而允许更多功能和问题修复。
    当它接受了代币的所有权，它会成为代币的唯一管理器，执行各个功能。

    如果想要升级管理器，我们需要把权限和相关数据转给一个新的管理器。

    智能代币只能在构建时设置，并且不可修改。
    为了更好的调用，代币的每个功能都被包裹了一下。

    注意：管理器可能把权限转交给一个没有管理功能的管理器，从而失去了升级管理器的能力
*/
contract SmartTokenController is TokenHolder {
    ISmartToken public token;   // 智能代币

    /**
        @dev 构建函数
    */
    function SmartTokenController(ISmartToken _token)
        public
        validAddress(_token)
    {
        token = _token;
    }

    // 确认控制器是代币的主人
    modifier active() {
        assert(token.owner() == address(this));
        _;
    }

    // 确认控制器不是代币的主人
    modifier inactive() {
        assert(token.owner() != address(this));
        _;
    }

    /**
        @dev 允许转换代币的所有权
        新的主人需要接受邀请
        只能被协约主人调用

        @param _newOwner    新的协约主人
    */
    function transferTokenOwnership(address _newOwner) public ownerOnly {
        token.transferOwnership(_newOwner);
    }

    /**
        @dev 新的主人接收代币
        只能被协约主人调用
    */
    function acceptTokenOwnership() public ownerOnly {
        token.acceptOwnership();
    }

    /**
        @dev 开始/关闭代币交易
        只能被协约主人调用

        @param _disable    true是关闭；false是开启
    */
    function disableTokenTransfers(bool _disable) public ownerOnly {
        token.disableTransfers(_disable);
    }

    /**
        @dev 提取控制器的代币到指定账号
        只能被主人调用

        @param _token   ERC20 代币协约地址
        @param _to      接受代币的地址
        @param _amount  提取数量
    */
    function withdrawFromToken(
        IERC20Token _token,
        address _to,
        uint256 _amount
    )
        public
        ownerOnly
    {
        ITokenHolder(token).withdrawTokens(_token, _to, _amount);
    }
}

/*
    Bancor Formula接口，计算代币转换价格
*/
contract IBancorFormula {
    function calculatePurchaseReturn(uint256 _supply, uint256 _connectorBalance, uint32 _connectorWeight, uint256 _depositAmount) public view returns (uint256);
    function calculateSaleReturn(uint256 _supply, uint256 _connectorBalance, uint32 _connectorWeight, uint256 _sellAmount) public view returns (uint256);
    function calculateCrossConnectorReturn(uint256 _connector1Balance, uint32 _connector1Weight, uint256 _connector2Balance, uint32 _connector2Weight, uint256 _amount) public view returns (uint256);
}

/*
    Bancor Gas Price Limit接口
*/
contract IBancorGasPriceLimit {
    function gasPrice() public view returns (uint256) {}
    function validateGasPrice(uint256) public view;
}

/*
    Bancor Quick Converter接口
*/
contract IBancorQuickConverter {
    function convert(IERC20Token[] _path, uint256 _amount, uint256 _minReturn) public payable returns (uint256);
    function convertFor(IERC20Token[] _path, uint256 _amount, uint256 _minReturn, address _for) public payable returns (uint256);
    function convertForPrioritized(IERC20Token[] _path, uint256 _amount, uint256 _minReturn, address _for, uint256 _block, uint256 _nonce, uint8 _v, bytes32 _r, bytes32 _s) public payable returns (uint256);
}

/*
    Bancor Converter Extensions接口
*/
contract IBancorConverterExtensions {
    function formula() public view returns (IBancorFormula) {}
    function gasPriceLimit() public view returns (IBancorGasPriceLimit) {}
    function quickConverter() public view returns (IBancorQuickConverter) {}
}

/*
    EIP228 Token Converter接口
*/
contract ITokenConverter {
    function convertibleTokenCount() public view returns (uint16);
    function convertibleToken(uint16 _tokenIndex) public view returns (address);
    function getReturn(IERC20Token _fromToken, IERC20Token _toToken, uint256 _amount) public view returns (uint256);
    function convert(IERC20Token _fromToken, IERC20Token _toToken, uint256 _amount, uint256 _minReturn) public returns (uint256);
    // deprecated, backward compatibility
    function change(IERC20Token _fromToken, IERC20Token _toToken, uint256 _amount, uint256 _minReturn) public returns (uint256);
}

/*
    Bancor Converter v0.8

    代币转换器，允许一个智能代币和其他代币之间的转换

    ERC20连接器的余额可以是虚拟的，从而不需要依赖于真实的余额，这有助于避免在一个协约中有大量金额的风险

    转换器可以升级

    警告：不建议对精度不足小数点后小于8位的智能代币使用该功能，因为可能造成精度的损失

    当前问题：
    - 以下机制缓解了前端攻击
        - minimum return 参数来定义交易的min/max价格
        - gas price limit 规避用户控制执行的顺序
      其他潜在的方案可能包含commit/reveal的机制
    - 考虑增加getters，从而客户不再需要以来结构题的参数排序
*/
contract BancorConverter is ITokenConverter, SmartTokenController, Managed {
    uint32 private constant MAX_WEIGHT = 1000000;
    uint32 private constant MAX_CONVERSION_FEE = 1000000;

    struct Connector {
        uint256 virtualBalance;         // connector 虚拟余额
        uint32 weight;                  // connector 权重，表示为ppm, 1-1000000
        bool isVirtualBalanceEnabled;   // 是否是虚拟余额
        bool isPurchaseEnabled;         // 能够购买代币
        bool isSet;                     // 映射元素是否定义了
    }

    string public version = '0.8';
    string public converterType = 'bancor';

    IBancorConverterExtensions public extensions;       // bancor converter extensions 协议
    IERC20Token[] public connectorTokens;               // ERC20 标准协议地址
    IERC20Token[] public quickBuyPath;                  // 用ETH购买token的方法
    mapping (address => Connector) public connectors;   // connector token addresses -> connector data
    uint32 private totalConnectorWeight = 0;            // 避免权重超过100%
    uint32 public maxConversionFee = 0;                 // 在协议周期内的最大转换费，表示为ppm, 0...1000000 (0 = no fee, 100 = 0.01%, 1000000 = 100%)
    uint32 public conversionFee = 0;                    // 当前的转换费，表示为pm, 0...maxConversionFee
    bool public conversionsEnabled = true;              // 转换是否开启
    IERC20Token[] private convertPath;

    // 当两个代币被转换时触发 (TokenConverter event)
    event Conversion(address indexed _fromToken, address indexed _toToken, address indexed _trader, uint256 _amount, uint256 _return,
                     int256 _conversionFee, uint256 _currentPriceN, uint256 _currentPriceD);
    // triggered when the conversion fee is updated
    event ConversionFeeUpdate(uint32 _prevFee, uint32 _newFee);

    /**
        @dev 构建函数

        @param  _token              转换者管理的智能代币
        @param  _extensions         bancor converter extensions 协约的地址
        @param  _maxConversionFee   最大的转换费，表示为ppm
        @param  _connectorToken     可选, 初始连接器，允许在指定时间进行部署
        @param  _connectorWeight    可选，初始连接器的权重
    */
    function BancorConverter(ISmartToken _token, IBancorConverterExtensions _extensions, uint32 _maxConversionFee, IERC20Token _connectorToken, uint32 _connectorWeight)
        public
        SmartTokenController(_token)
        validAddress(_extensions)
        validMaxConversionFee(_maxConversionFee)
    {
        extensions = _extensions;
        maxConversionFee = _maxConversionFee;

        if (_connectorToken != address(0))
            addConnector(_connectorToken, _connectorWeight, false);
    }

    // 验证连接器的代币地址 —— 验证地址是否属于一个连接器的代币
    modifier validConnector(IERC20Token _address) {
        require(connectors[_address].isSet);
        _;
    }

    // 验证代币的地址 —— 验证一个地址属于可转换的代币
    modifier validToken(IERC20Token _address) {
        require(_address == token || connectors[_address].isSet);
        _;
    }

    // 验证最大的累计转换费
    modifier validMaxConversionFee(uint32 _conversionFee) {
        require(_conversionFee >= 0 && _conversionFee <= MAX_CONVERSION_FEE);
        _;
    }

    // 验证转换费
    modifier validConversionFee(uint32 _conversionFee) {
        require(_conversionFee >= 0 && _conversionFee <= maxConversionFee);
        _;
    }

    // 验证连接器权重范围
    modifier validConnectorWeight(uint32 _weight) {
        require(_weight > 0 && _weight <= MAX_WEIGHT);
        _;
    }

    // 验证连接器的路径 —— 验证元素的个数大于2并且最大的条数小于10
    modifier validConversionPath(IERC20Token[] _path) {
        require(_path.length > 2 && _path.length <= (1 + 2 * 10) && _path.length % 2 == 1);
        _;
    }

    // 验证转换被允许
    modifier conversionsAllowed {
        assert(conversionsEnabled);
        _;
    }

    // 只允许主人或者管理员操作
    modifier ownerOrManagerOnly {
        require(msg.sender == owner || msg.sender == manager);
        _;
    }

    // 只允许快速转换的主人执行
    modifier quickConverterOnly {
        require(msg.sender == address(extensions.quickConverter()));
        _;
    }

    /**
        @dev 返回定义的token数量

        @return 定义的token数量
    */
    function connectorTokenCount() public view returns (uint16) {
        return uint16(connectorTokens.length);
    }

    /**
        @dev 返回该协议支持的可转换的代币的数量
        注意可转换的代币的数量是连接器的代币数量加一（表示智能代币）

        @return 可转换的代币的数量
    */
    function convertibleTokenCount() public view returns (uint16) {
        return connectorTokenCount() + 1;
    }

    /**
        @dev 基于一个可转换的代币的索引，返回协议的地址

        @param _tokenIndex  可转换的代币的索引

        @return convertible 协议的地址
    */
    function convertibleToken(uint16 _tokenIndex) public view returns (address) {
        if (_tokenIndex == 0)
            return token;
        return connectorTokens[_tokenIndex - 1];
    }

    /*
        @dev allows 允许主人更新扩展协约的地址

        @param _extensions    bancor converter extensions contract的地址
    */
    function setExtensions(IBancorConverterExtensions _extensions)
        public
        ownerOnly
        validAddress(_extensions)
        notThis(_extensions)
    {
        extensions = _extensions;
    }

    /*
        @dev 允许管理员更新快速购买通道

        @param _path    新的快速购买通道，查看BancorQuickConverter协约中的快速购买通道
    */
    function setQuickBuyPath(IERC20Token[] _path)
        public
        ownerOnly
        validConversionPath(_path)
    {
        quickBuyPath = _path;
    }

    /*
        @dev 允许管理员清楚快速购买通道
    */
    function clearQuickBuyPath() public ownerOnly {
        quickBuyPath.length = 0;
    }

    /**
        @dev 返回快速购买通道的长度

        @return 快速购买通道的长度
    */
    function getQuickBuyPathLength() public view returns (uint256) {
        return quickBuyPath.length;
    }

    /**
        @dev 关闭整个转换功能
        这是紧急情况下的应对方案
        只有管理员能操作

        @param _disable 是否关闭
    */
    function disableConversions(bool _disable) public ownerOrManagerOnly {
        conversionsEnabled = !_disable;
    }

    /**
        @dev 更新当前的转换费用
        只能被管理员调用

        @param _conversionFee新的转换费，表示为ppm
    */
    function setConversionFee(uint32 _conversionFee)
        public
        ownerOrManagerOnly
        validConversionFee(_conversionFee)
    {
        ConversionFeeUpdate(conversionFee, _conversionFee);
        conversionFee = _conversionFee;
    }

    /*
        @dev 返回一个数量对应的转换费

        @return 转换费的数量
    */
    function getConversionFeeAmount(uint256 _amount) public view returns (uint256) {
        return safeMul(_amount, conversionFee) / MAX_CONVERSION_FEE;
    }

    /**
        @dev 为代币定义一个新的连接器
        只有在转换器关闭的情况下被主人调用

        @param _token                  连接器代币的地址
        @param _weight                 常数连接器权重，表示为ppm, 1-1000000
        @param _enableVirtualBalance   是否允许虚拟余额
    */
    function addConnector(IERC20Token _token, uint32 _weight, bool _enableVirtualBalance)
        public
        ownerOnly
        inactive
        validAddress(_token)
        notThis(_token)
        validConnectorWeight(_weight)
    {
        require(_token != token && !connectors[_token].isSet && totalConnectorWeight + _weight <= MAX_WEIGHT); // validate input

        connectors[_token].virtualBalance = 0;
        connectors[_token].weight = _weight;
        connectors[_token].isVirtualBalanceEnabled = _enableVirtualBalance;
        connectors[_token].isPurchaseEnabled = true;
        connectors[_token].isSet = true;
        connectorTokens.push(_token);
        totalConnectorWeight += _weight;
    }

    /**
        @dev 更新一个代币连接器
        只能被拥有者调用

        @param _connectorToken         连接器代币的地址
        @param _weight                 常数连接器权重，表示为ppm, 1-1000000
        @param _enableVirtualBalance   是否允许虚拟余额
        @param _virtualBalance         新的连接器的虚拟余额
    */
    function updateConnector(IERC20Token _connectorToken, uint32 _weight, bool _enableVirtualBalance, uint256 _virtualBalance)
        public
        ownerOnly
        validConnector(_connectorToken)
        validConnectorWeight(_weight)
    {
        Connector storage connector = connectors[_connectorToken];
        require(totalConnectorWeight - connector.weight + _weight <= MAX_WEIGHT); // 警告（fennec）：这里可能有溢出风险

        totalConnectorWeight = totalConnectorWeight - connector.weight + _weight;
        connector.weight = _weight;
        connector.isVirtualBalanceEnabled = _enableVirtualBalance;
        connector.virtualBalance = _virtualBalance;
    }

    /**
        @dev 关闭一个连接器代币的购买能力，用来预防危险
        只能被拥有者调用
        请注意：销售行为依然被允许了

        @param _connectorToken  连接器代币协议地址
        @param _disable         是否关闭购买
    */
    function disableConnectorPurchases(IERC20Token _connectorToken, bool _disable)
        public
        ownerOnly
        validConnector(_connectorToken)
    {
        connectors[_connectorToken].isPurchaseEnabled = !_disable;
    }

    /**
        @dev returns 返回连接器的余额

        @param _connectorToken  连接器代币协议地址

        @return 连接器的余额
    */
    function getConnectorBalance(IERC20Token _connectorToken)
        public
        view
        validConnector(_connectorToken)
        returns (uint256)
    {
        Connector storage connector = connectors[_connectorToken];
        return connector.isVirtualBalanceEnabled ? connector.virtualBalance : _connectorToken.balanceOf(this);
    }

    /**
        @dev 返回从一个代币转换为另一个代币的预期数量

        @param _fromToken  ERC20 被转换的代币
        @param _toToken    ERC20 转换成的代币
        @param _amount     转换的数量

        @return 与其转换的数量
    */
    function getReturn(IERC20Token _fromToken, IERC20Token _toToken, uint256 _amount) public view returns (uint256) {
        require(_fromToken != _toToken); // 验证输入

        // 基于当前代币转换
        if (_toToken == token)
            return getPurchaseReturn(_fromToken, _amount);
        else if (_fromToken == token)
            return getSaleReturn(_toToken, _amount);

        // 在两个连接器之间转换
        uint256 purchaseReturnAmount = getPurchaseReturn(_fromToken, _amount);
        return getSaleReturn(_toToken, purchaseReturnAmount, safeAdd(token.totalSupply(), purchaseReturnAmount));
    }

    /**
        @dev 返回通过一个连接器代币购买代币的预期结果

        @param _connectorToken  连接器代币协约地址
        @param _depositAmount   买入的数量

        @return 预期的数量
    */
    function getPurchaseReturn(IERC20Token _connectorToken, uint256 _depositAmount)
        public
        view
        active
        validConnector(_connectorToken)
        returns (uint256)
    {
        Connector storage connector = connectors[_connectorToken];
        require(connector.isPurchaseEnabled); // validate input

        uint256 tokenSupply = token.totalSupply();
        uint256 connectorBalance = getConnectorBalance(_connectorToken);
        uint256 amount = extensions.formula().calculatePurchaseReturn(tokenSupply, connectorBalance, connector.weight, _depositAmount);

        // 扣除费用
        uint256 feeAmount = getConversionFeeAmount(amount);
        return safeSub(amount, feeAmount);
    }

    /**
        @dev 返回通过一个连接器代币卖出代币的预期结果

        @param _connectorToken  连接器代币协约地址
        @param _sellAmount      卖出的数量

        @return 预期得到的数量
    */
    function getSaleReturn(IERC20Token _connectorToken, uint256 _sellAmount) public view returns (uint256) {
        return getSaleReturn(_connectorToken, _sellAmount, token.totalSupply());
    }

    /**
        @dev 从_fromToken 转化为 _toToken

        @param _fromToken  用来转换ERC20代币
        @param _toToken    被转换到的ERC20代币
        @param _amount     转换的数量，基于fromToken
        @param _minReturn  限制转换的结果需要高于minReturn，否则取消

        @return conversion 返回数量
    */
    function convertInternal(IERC20Token _fromToken, IERC20Token _toToken, uint256 _amount, uint256 _minReturn) public quickConverterOnly returns (uint256) {
        require(_fromToken != _toToken); // 验证输入

        // 在代币和连接器见转换
        if (_toToken == token)
            return buy(_fromToken, _amount, _minReturn);
        else if (_fromToken == token)
            return sell(_toToken, _amount, _minReturn);

        // 在两个连接器间转换
        uint256 purchaseAmount = buy(_fromToken, _amount, 1);
        return sell(_toToken, purchaseAmount, _minReturn);
    }

    /**
        @dev 将一定数量的_fromToken 转换为 _toToken

        @param _fromToken  用来转换ERC20代币
        @param _toToken    被转换到的ERC20代币
        @param _amount     转换的数量，基于fromToken
        @param _minReturn  限制转换的结果需要高于minReturn，否则取消

        @return conversion 返回数量
    */
    function convert(IERC20Token _fromToken, IERC20Token _toToken, uint256 _amount, uint256 _minReturn) public returns (uint256) {
            convertPath = [_fromToken, token, _toToken];
            return quickConvert(convertPath, _amount, _minReturn);
    }

    /**
        @dev 通过放入连接器代币来购买代币

        @param _connectorToken  连接器代币协议地址
        @param _depositAmount   存入的数量，基于连接器代币
        @param _minReturn       限制转换的结果需要高于minReturn，否则取消

        @return 返回数量
    */
    function buy(IERC20Token _connectorToken, uint256 _depositAmount, uint256 _minReturn)
        internal
        conversionsAllowed
        greaterThanZero(_minReturn)
        returns (uint256)
    {
        uint256 amount = getPurchaseReturn(_connectorToken, _depositAmount);
        require(amount != 0 && amount >= _minReturn); // 确定有返回结果，并且不小于阈值

        // 更新相关的虚拟余额
        Connector storage connector = connectors[_connectorToken];
        if (connector.isVirtualBalanceEnabled)
            connector.virtualBalance = safeAdd(connector.virtualBalance, _depositAmount);

        // 从调用者处转换代币进来
        assert(_connectorToken.transferFrom(msg.sender, this, _depositAmount));
        // 给调用者发布新的代币
        token.issue(msg.sender, amount);

        dispatchConversionEvent(_connectorToken, _depositAmount, amount, true);
        return amount;
    }

    /**
        @dev 通过提取连接器代币来卖掉代币

        @param _connectorToken  连接器代币协约地址
        @param _sellAmount      卖的数量，基于智能代币
        @param _minReturn       限制转换的结果需要高于minReturn，否则取消

        @return 返回数量
    */
    function sell(IERC20Token _connectorToken, uint256 _sellAmount, uint256 _minReturn)
        internal
        conversionsAllowed
        greaterThanZero(_minReturn)
        returns (uint256)
    {
        require(_sellAmount <= token.balanceOf(msg.sender)); // 验证输入

        uint256 amount = getSaleReturn(_connectorToken, _sellAmount);
        require(amount != 0 && amount >= _minReturn); // 确定有返回结果，并且不小于阈值

        uint256 tokenSupply = token.totalSupply();
        uint256 connectorBalance = getConnectorBalance(_connectorToken);
        // 确认交易在供应量耗尽的情况下，只会耗尽该连接器
        assert(amount < connectorBalance || (amount == connectorBalance && _sellAmount == tokenSupply));

        // 在需要时更新虚拟余额
        Connector storage connector = connectors[_connectorToken];
        if (connector.isVirtualBalanceEnabled)
            connector.virtualBalance = safeSub(connector.virtualBalance, amount);

        // 把调用者卖掉的_sellAmount的代币销毁
        token.destroy(msg.sender, _sellAmount);
        // 把代币转回调用者
        // 如果实际的金额小于虚拟金额，则可能会失败
        assert(_connectorToken.transfer(msg.sender, amount));

        dispatchConversionEvent(_connectorToken, _sellAmount, amount, false);
        return amount;
    }

    /**
        @dev 通过之前定义的转换路径来转换代币
        注意：当从ERC20代币进行转换，需要提前设置补贴

        @param _path        转换路径
        @param _amount      转换的数量
        @param _minReturn   限制转换的结果需要高于minReturn，否则取消

        @return 返回数量
    */
    function quickConvert(IERC20Token[] _path, uint256 _amount, uint256 _minReturn)
        public
        payable
        validConversionPath(_path)
        returns (uint256)
    {
        return quickConvertPrioritized(_path, _amount, _minReturn, 0x0, 0x0, 0x0, 0x0, 0x0);
    }

    /**
        @dev 通过之前定义的转换路径来转换代币
        注意：当从ERC20代币进行转换，需要提前设置补贴

        @param _path        转换路径
        @param _amount      转换的数量
        @param _minReturn   限制转换的结果需要高于minReturn，否则取消
        @param _block       如果当前的区块超过了参数，则取消
        @param _nonce       发送者地址的nonce
        @param _v           通过交易签名提取
        @param _r           通过交易签名提取
        @param _s           通过交易签名提取

        @return 返回数量
    */
    function quickConvertPrioritized(IERC20Token[] _path, uint256 _amount, uint256 _minReturn, uint256 _block, uint256 _nonce, uint8 _v, bytes32 _r, bytes32 _s)
        public
        payable
        validConversionPath(_path)
        returns (uint256)
    {
        IERC20Token fromToken = _path[0];
        IBancorQuickConverter quickConverter = extensions.quickConverter();

        // 我们需要从调用者向快速转换着把源代币转化
        // 因此他能基于调用者进行转换
        if (msg.value == 0) {
            // 如果不是ETH，把源代币发给快速调用者
            // 如果是智能代币，不需要补贴 —— 销毁代币，然后发给快速转换者
            if (fromToken == token) {
                token.destroy(msg.sender, _amount); // 销毁调用者的_amount代币
                token.issue(quickConverter, _amount); // 把_amount的新代币发给快速转换者
            } else {
                // 否则，我们假设有了补贴，发给快速转换者
                assert(fromToken.transferFrom(msg.sender, quickConverter, _amount));
            }
        }

        // 执行转换，把ETH转回
        return quickConverter.convertForPrioritized.value(msg.value)(_path, _amount, _minReturn, msg.sender, _block, _nonce, _v, _r, _s);
    }

    // 弃用了，向后兼容
    function change(IERC20Token _fromToken, IERC20Token _toToken, uint256 _amount, uint256 _minReturn) public returns (uint256) {
        return convertInternal(_fromToken, _toToken, _amount, _minReturn);
    }

    /**
        @dev 工具，基于一个总供应量，返回基于一个连接器代币来卖掉代币的期待返回

        @param _connectorToken  连接器代币协议地址
        @param _sellAmount      销售的数量
        @param _totalSupply     设置总供应量

        @return 返回的数量
    */
    function getSaleReturn(IERC20Token _connectorToken, uint256 _sellAmount, uint256 _totalSupply)
        private
        view
        active
        validConnector(_connectorToken)
        greaterThanZero(_totalSupply)
        returns (uint256)
    {
        Connector storage connector = connectors[_connectorToken];
        uint256 connectorBalance = getConnectorBalance(_connectorToken);
        uint256 amount = extensions.formula().calculateSaleReturn(_totalSupply, connectorBalance, connector.weight, _sellAmount);

        // 从返回的数量中剪掉费用
        uint256 feeAmount = getConversionFeeAmount(amount);
        return safeSub(amount, feeAmount);
    }

    /**
        @dev 辅助函数，分发转换事件
        这个函数在计算当前价格时也会考虑到代币的小数位

        @param _connectorToken  连接器代币的协约地址
        @param _amount          购买/出售的数量
        @param _returnAmount    返回的数量
        @param isPurchase       是购买还是出售
    */
    function dispatchConversionEvent(IERC20Token _connectorToken, uint256 _amount, uint256 _returnAmount, bool isPurchase) private {
        Connector storage connector = connectors[_connectorToken];

        // 用简单的方法计算价格
        // price = connector balance / (supply * weight)
        // 权重表示为，ppm，因此乘以1000000
        uint256 connectorAmount = safeMul(getConnectorBalance(_connectorToken), MAX_WEIGHT);
        uint256 tokenAmount = safeMul(token.totalSupply(), connector.weight);

        // normalize values
        uint8 tokenDecimals = token.decimals();
        uint8 connectorTokenDecimals = _connectorToken.decimals();
        if (tokenDecimals != connectorTokenDecimals) {
            if (tokenDecimals > connectorTokenDecimals)
                connectorAmount = safeMul(connectorAmount, 10 ** uint256(tokenDecimals - connectorTokenDecimals));
            else
                tokenAmount = safeMul(tokenAmount, 10 ** uint256(connectorTokenDecimals - tokenDecimals));
        }

        uint256 feeAmount = getConversionFeeAmount(_returnAmount);
        // 确认费用不超过255个bit，从而避免溢出
        assert(feeAmount <= 2 ** 255);

        if (isPurchase)
            Conversion(_connectorToken, token, msg.sender, _amount, _returnAmount, int256(feeAmount), connectorAmount, tokenAmount);
        else
            Conversion(token, _connectorToken, msg.sender, _amount, _returnAmount, int256(feeAmount), tokenAmount, connectorAmount);
    }

    /**
        @dev 另一种方法，用ETH购买智能代币
        注意：使用购买时的架构来买
    */
    function() payable public {
        quickConvert(quickBuyPath, msg.value, 1);
    }
}

```


## 总结

我们这里只是对Ethereum的架构进行了基本介绍，让我们在下一节课中继续深入探讨。

## 参考资料

- [Bancor](https://www.bancor.network)
