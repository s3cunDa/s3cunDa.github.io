# Eigenlayer

一年多前写的老东西了，仅作记录。

## Overview

Eigenlayer实现了一个Restaking的基础设施，为一些需要链下计算需要解决的共识信任问题提供了一个解决方案。对于普通用户，这套方案将用户质押的闲置资产进行了再次利用，赚取了更多的收益；对于预言机或者L2等需要进行链下计算验证的应用，提供了一个模块化的开发平台，降低了开发的门槛，同时不需要项目方额外进行代币生态的维护。

Eigenlayer是一个restaking项目，它实现了一个自由的restaking平台，允许用户将信标链质押的ETH、原生ETH、LST等资产作为质押物进行再质押。进行再质押的用户可以进一步选择性的将再质押资产质押到平台上的应用，以获取更多的奖励金。

目前，对于一些需要共识的应用场景，如跨链桥、预言机，项目方需要自己构建一套完整的验证体系。如果项目方使用PoS作为某一个应用的共识验证方案，那么项目方需要自己构建一套链下的验证服务，这套服务可能需要发行新的代币作为质押资产，同时也需要一定的开发成本。

上述场景面临如下问题：

1. 开发验证服务（AVS）成本高。
2. 需要维护代币生态
3. 共识体系被攻击风险高

为了解决上述问题，Eigenlayer提出了两个新颖的方案：通过再质押实现的共同安全保证（Pooled security via restaking）以及自由市场管理机制（free-market governance）。

*Pooled security via restaking & Open marketplace.*

Eigenlayer允许构建自定义的模块，这些模块会被再质押的ETH作为质押资产进行安全性保障。对于质押在以太坊信标链的资产，用户可以将质押在信标链的ETH的取款凭证地址设置为Eigenlayer合约地址，以此来进行再质押。除此之外，Eigenlayer也支持原生ETH和其它种LST或者LST lp等形式的资产的质押。

Eigenlayer构建了一个开放市场，管理各个模块所对应的质押代币的数量以及AVS的奖惩机制。对于模块的管理者，需要有明确的激励来吸引用户参与其中。再质押用户可以选择成为validator参与到Eigenlayer的模块上，此时用户需要参与到模块的共识验证中。用户同时也可以将质押的资产委托给某个验证者，本身不参与验证仅提供资产拿到分红。

解决了什么问题：

1. 开发AVS成本：Eigenlayer自己实现了一个cli，开发者可以进行二次开发，并且共识的数据存储在ETH上，可信度高。
2. 代币生态：Eigenlayer的质押代币与ETH深层绑定，出现价格脱锚风险低。
3. 信任集成：使用restaking，被攻击的风险被转移到了ETH上。
4. 闲置资产利用

## 实现

主要参考：https://github.com/Layr-Labs/eigenlayer-contracts/tree/master

### Restaking

ETH from beacon & Native ETH：由EigenPods实现：https://github.com/Layr-Labs/eigenlayer-contracts/blob/master/src/contracts/pods

对于每一个staker，创建单独的pods合约进行记录。用户可以通过pod将原生ETH质押到信标链，同时信标链中质押的ETH，通过验证stateroot、withdrawroot的方式进行再质押，值得注意的是，beaconchain的stateroot由外部预言机进行维护更新。

LST等：

入口为manager合约：https://github.com/Layr-Labs/eigenlayer-contracts/blob/master/src/contracts/core/StrategyManager.sol

由strategies实现，白名单模式，简单的underlying、shares模式。https://github.com/Layr-Labs/eigenlayer-contracts/blob/master/src/contracts/strategies

### Delegation

https://github.com/Layr-Labs/eigenlayer-contracts/blob/master/src/contracts/core/DelegationManager.sol

注册成为Operator，可以选择将全部质押资产委托给某一Operator，委托时需要委托者与被委托者进行双向确认。

委托关系可以随时取消，并且没有看到slash相关的操作。

### Slashing

https://github.com/Layr-Labs/eigenlayer-contracts/blob/master/src/contracts/core/Slasher.sol

主要逻辑均为链下中间件服务，链上合约中没有体现slash的逻辑（可能未实现）。

需要Operator注册对应的slasher地址，由slasher来决定是否对某个 Operator进行slash。

master分支只实现了一些简单的操作，没有实现完整的逻辑。