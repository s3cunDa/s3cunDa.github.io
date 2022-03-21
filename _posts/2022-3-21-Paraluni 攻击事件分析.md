# Paraluni 攻击事件分析

## 事件描述

Paraluni （平行宇宙）是新加坡 Parallel Universe 基金会发布的一个 基于币安智能链的 DeFi 项目。在 2022 年 03 月 13 日，Paraluni 遭受黑客攻击，损失约 170 万美元。

## 攻击流程分析

交易：https://dashboard.tenderly.co/tx/bsc/0x70f367b9420ac2654a5223cc311c7f9c361736a39fd4e7dff9ed1b85bab7ad54

攻击者首先创建了两个token，分别为UGT（russia good token）、UBT（Ukraine bad token）。其中，UBT代币的transferfrom被改写，会调用masterchef的deposit函数。

攻击者闪电贷借出了156,984 BSC-USD和157,210 BUSD。

随后，将这两个借出的代币存入对应的parapair（https://bscscan.com/address/0x3fd4fbd7a83062942b6589a2e9e2436dd8e134d4#code），换取LP代币。并将LP代币存入UBT合约地址。

到此，前期的准备工作完成。

攻击者通过调用masterchef的depositByAddLiquidity函数来进行此次重入攻击。

```solidity
    function depositByAddLiquidity(uint256 _pid, address[2] memory _tokens, uint256[2] memory _amounts) external{
        require(_amounts[0] > 0 && _amounts[1] > 0, "!0");
        address[2] memory tokens;
        uint256[2] memory amounts;
        (tokens[0], amounts[0]) = _doTransferIn(msg.sender, _tokens[0], _amounts[0]);
        (tokens[1], amounts[1]) = _doTransferIn(msg.sender, _tokens[1], _amounts[1]);
        depositByAddLiquidityInternal(msg.sender, _pid, tokens,amounts);
    }
```

调用的参数如下：

```
{
  "_pid": "18",
  "_tokens": [
    "0xbc5db89ce5ab8035a71c6cd1cd0f0721ad28b508",
    "0xca2ca459ec6e4f58ad88aeb7285d2e41747b9134"
  ],
  "_amounts": [
    "1000000000000000000",
    "1000000000000000000"
  ]
}
```

其中，pid18对应的是之前BSC-USD和BUSD的交易对，两个token地址分别为UGT和UBTtoken地址。（看起来很奇怪，把两个不相关的token存进别的交易池，这里先留白，后面会讲）。

其中_doTransferIn函数会讲对应数量的token转到masterchef，其底层是调用safeTransferFrom函数。

这里tenderly有解析错误

![image-20220321221828371](https://s3cunDa.github.io/assets/post/image-20220321221828371.png)

而0x23b872dd是transferFrom的选择子（这里还有个冲突，感觉可以整个ctf题）：

![image-20220321221934719](https://s3cunDa.github.io/assets/post/image-20220321221934719.png)

不过在这里，攻击者预先设置的transferFrom函数没有调用deposit，因为它内置了一个对caller的检查：

![image-20220321222301635](https://s3cunDa.github.io/assets/post/image-20220321222301635.png)

所以漏洞利用并不是在这一步。

随后，执行完了转token操作后，就会调用depositByAddLiquidityInternal函数，其逻辑如下：

```solidity
    function depositByAddLiquidityInternal(address _user, uint256 _pid, address[2] memory _tokens, uint256[2] memory _amounts) internal {
        PoolInfo memory pool = poolInfo[_pid];
        require(address(pool.ticket) == address(0), "T:E");
        uint liquidity = addLiquidityInternal(address(pool.lpToken), _user, _tokens, _amounts);
        _deposit(_pid, liquidity, _user);
    }
```

可以看到，主要逻辑就是调用addLiquidityInternal将流动性加到池子中，计算流动性，然后调用_deposit。

而addLiquidityInternal逻辑如下：

```solidity
    function addLiquidityInternal(address _lpAddress, address _user, address[2] memory _tokens, uint256[2] memory _amounts) internal returns (uint){
        //Stack too deep, try removing local variables
        DepositVars memory vars;
        approveIfNeeded(_tokens[0], address(paraRouter), _amounts[0]);
        approveIfNeeded(_tokens[1], address(paraRouter), _amounts[1]);
        vars.oldBalance = IERC20(_lpAddress).balanceOf(address(this));
        (vars.amountA, vars.amountB, vars.liquidity) = paraRouter.addLiquidity(_tokens[0], _tokens[1], _amounts[0], _amounts[1], 1, 1, address(this), block.timestamp + 600);
        vars.newBalance = IERC20(_lpAddress).balanceOf(address(this));
        require(vars.newBalance > vars.oldBalance, "B:E");
        vars.liquidity = vars.newBalance.sub(vars.oldBalance);
        addChange(_user, _tokens[0], _amounts[0].sub(vars.amountA));
        addChange(_user, _tokens[1], _amounts[1].sub(vars.amountB));
        return vars.liquidity;
    }
```

也是看起来很美好的通过前后余额来判断增加的流动性（另外，这里paraRouter.addLiquidity返回的流动性就是个摆设）增加流动性的操作在paraRouter.addLiquidity实现。

```solidity
    function addLiquidity(
        address tokenA,
        address tokenB,
        uint amountADesired,
        uint amountBDesired,
        uint amountAMin,
        uint amountBMin,
        address to,
        uint deadline
    ) external virtual override ensure(deadline) returns (uint amountA, uint amountB, uint liquidity) {
        noFees(tokenA, tokenB);
        (amountA, amountB) = _addLiquidity(tokenA, tokenB, amountADesired, amountBDesired, amountAMin, amountBMin);
        address pair = ParaLibrary.pairFor(factory, tokenA, tokenB);
        TransferHelper.safeTransferFrom(tokenA, msg.sender, pair, amountA);
        TransferHelper.safeTransferFrom(tokenB, msg.sender, pair, amountB);
        liquidity = IParaPair(pair).mint(to);
        FeesOn(tokenA, tokenB);
    }
```

addLiquidity通过两个token的地址找到对应的pair，并没有采用pid的方式寻找，也就是说，在这里计算的流动性是攻击者创建的两个伪造token的pair，按理说是过不了addLiquidityInternal的流动性增加check的。毕竟毫不相关的两个pair。（计算出的流动性也是攻击者创建的那个pair，最后返回的流动性在上一个调用函数里面根本没用到。）

那么这里就用到了之前准备工作中的UBTtoken的魔改transferFrom函数了。

攻击者创建的transferFrom函数会调用masterchef的deposit函数：

```solidity
    function deposit(uint256 _pid, uint256 _amount) external {
        depositInternal(_pid, _amount, msg.sender, msg.sender);
    }
    function depositInternal(uint256 _pid, uint256 _amount, address _user, address payer) internal {
        PoolInfo storage pool = poolInfo[_pid];
        pool.lpToken.safeTransferFrom(
            address(payer),
            address(this),
            _amount
        );
        if (address(pool.ticket) != address(0)) {
            UserInfo storage user = userInfo[_pid][_user];
            uint256 new_amount = user.amount.add(_amount);
            uint256 user_ticket_count = pool.ticket.tokensOfOwner(_user).length;
            uint256 staked_ticket_count = ticket_staked_count(_user, address(pool.ticket));
            uint256 ticket_level = pool.ticket.level();
            (, uint overflow) = check_vip_limit(ticket_level, user_ticket_count + staked_ticket_count, new_amount);
            require(overflow == 0, "Exceeding the ticket limit");
            deposit_all_tickets(pool.ticket);
        }
        _deposit(_pid, _amount, _user);
    }
```

其calldata如下：

```json
{
  "_pid": "18",
  "_amount": "155935695765009852802486",
  "_user": "0xca2ca459ec6e4f58ad88aeb7285d2e41747b9134",
  "payer": "0xca2ca459ec6e4f58ad88aeb7285d2e41747b9134"
}
```

也就是将之前准备工作的所有LP都存入masterchef。

也就是说，刚才提到的一切工作都是无关痛痒，真正的存储逻辑在这里。

执行完了deposit函数之后，masterchef的余额增加，自然而然就能过了addLiquidityInternal中的检查了。

所以说攻击者曲线救国，用设计缺陷搞了这么个逻辑，是为什么呢？不是偷钱嘛？

现在我们回到deposit函数和depositByAddLiquidityInternal函数逻辑，他们最终都会调用_deposit函数：

```solidity
    function _deposit(uint256 _pid, uint256 _amount, address _user) internal {
        PoolInfo storage pool = poolInfo[_pid];
        UserInfo storage user = userInfo[_pid][_user];
        poolsTotalDeposit[_pid] = poolsTotalDeposit[_pid].add(_amount);
        updatePool(_pid);
        if (user.amount > 0) {
            uint256 pending =
                user.amount.mul(pool.accT42PerShare).div(1e12).sub(
                    user.rewardDebt
                );
            _claim(pool.pooltype, pending);
        }
        user.amount = user.amount.add(_amount);
        user.rewardDebt = user.amount.mul(pool.accT42PerShare).div(1e12);
        emit Deposit(_user, _pid, _amount);
    }
```

可以看到主要逻辑就是在对应的池子里和用户的账户里增加余额。

也就是说，deposit函数和depositByAddLiquidityInternal都会使得用户的余额增加，但是在整个交易序列中，用户只存了一次钱，这也就是本次重入攻击的最终目的和效果。

### 总结

本次攻击的涉及漏洞有三点：

1. 没有检查pid和token的对应性
2. 没有检查传入token的标准函数实现
3. 项目开发逻辑混乱，一会用pid，一会用token来定位pair，而且代码还有冗余，一致性差。

思考：

入行一个月，发现现在许多漏洞的成因都在于没有检查ERC20的标准接口实现，项目方总是想当然的认为ERC20token的实现合法，个人认为这个方向可以作为之后漏洞挖掘代码审计的一个抓手。

合约中的防御机制类似于安全带和安全气囊，项目的安全最终还是建立在开发者的代码逻辑完备上的，像本次项目这种代码逻辑混乱校验时有时无的情况无异于醉酒驾驶，出问题早晚的事。



