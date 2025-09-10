# Stacks节点工作流程

随着多次升级迭代，Stacks 节点代码包含了不同版本的逻辑。以下内容仅针对 Nakamoto 升级后的最新版本进行讨论。

Stacks 节点的代码结构由多个模块组成，核心入口为[ main](https://github.com/stacks-network/stacks-core/tree/master/testnet/stacks-node/src/main.rs#L268) 函数。main 函数随后会调用[ start](https://github.com/stacks-network/stacks-core/tree/master/testnet/stacks-node/src/run_loop/nakamoto.rs#L398) 函数，启动最新的 Nakamoto 运行主循环。

通过主循环的初始化逻辑，可以识别出 Stacks 节点的几个关键模块：

1. [BitcoinRegtestController](https://github.com/stacks-network/stacks-core/tree/master/testnet/stacks-node/src/burnchains/bitcoin_regtest_controller.rs#L72)：负责 BTC 区块的同步和与 BTC 的通信。
2. [PeerThread](https://github.com/stacks-network/stacks-core/tree/master/testnet/stacks-node/src/nakamoto_node/peer.rs#L42)：处理节点间的 P2P 通信。
3. [ChainsCoordinator](https://github.com/stacks-network/stacks-core/tree/master/stackslib/src/chainstate/coordinator/mod.rs)：负责处理 Stacks 区块。
4. [RelayerThread](https://github.com/stacks-network/stacks-core/tree/master/testnet/stacks-node/src/nakamoto_node/relayer.rs#L02)：管理共识的逻辑。
5. [BlockMinerThread](https://github.com/stacks-network/stacks-core/tree/master/testnet/stacks-node/src/nakamoto_node/miner.rs#L89)：负责出块逻辑（仅在节点运行者选择成为矿工时激活）。

以上各模块通过 globals 中的 channels 进行交互。例如，当 ChainsCoordinator 处理 Stacks 区块时，会根据 Relayer 和 BitcoinRegtestController 发来的数据进行相应的操作。具体实现可参考[ coordinator](https://github.com/stacks-network/stacks-core/tree/master/stackslib/src/chainstate/coordinator/comm.rs#L36) 模块的代码。

# PoX是如何工作的

在 Stacks 的 PoX 共识机制中引入了 **Tenure** 概念。与传统以区块为单位的机制不同，PoX 以时间为单位来定义矿工的任期。下面详细描述了从一个矿工的 Tenure 结束到下一个矿工的 Tenure 结束的 PoX 工作流程。

#### **主要的共识流程**

共识的核心工作流程始于 relayer 模块的 main 函数[ testnet/stacks-node/src/nakamoto_node/relayer.rs#L804](https://github.com/stacks-network/stacks-core/tree/master/testnet/stacks-node/src/nakamoto_node/relayer.rs#L804)。在这个循环中，每 10 秒检查以下内容：

1. **Burnchain 状态的变化**。
2. **VRF Key 注册状态**：[testnet/stacks-node/src/nakamoto_node/relayer.rs#L715](https://github.com/stacks-network/stacks-core/tree/master/testnet/stacks-node/src/nakamoto_node/relayer.rs#L715)。
3. **P2P 网络中的新信息**：[testnet/stacks-node/src/nakamoto_node/relayer.rs#L824](https://github.com/stacks-network/stacks-core/tree/master/testnet/stacks-node/src/nakamoto_node/relayer.rs#L824)。

根据这些变化，relayer 执行相应的指令（**Directives**）来推进共识流程[ testnet/stacks-node/src/nakamoto_node/relayer.rs#L847](https://github.com/stacks-network/stacks-core/tree/master/testnet/stacks-node/src/nakamoto_node/relayer.rs#L847)。

1. **HandleNetResult**：处理从 P2P 网络获取的数据（如 Stacks 区块、Stacks 交易等），BTC 区块在此阶段不涉及。
2. **RegisterKey**：注册 VRF 公钥。当节点选择成为矿工时，会在 relayer 中注册 VRF Key，这使矿工能够参与未来的出块选举。
3. **ProcessedBurnBlock**：当新的 BTC 区块被 SortitionDB 处理完毕后，系统会检查该区块中的矿工选举结果。如果当前节点在该 BTC 区块中被选为矿工，则触发出块逻辑。
4. **IssueBlockCommit**：用于竞选下一个 Tenure 的出块权。在新的 BTC 区块处理完毕或当前 Tenure 的第一个 Stacks 区块被确认后，relayer 会通知矿工竞选下一个 Tenure 的出块权。

1. 

为了成为 Stacks 的矿工，节点首先需要注册一个 **VRF 公钥**（Verifiable Random Function Key）。此注册使得该节点在未来的 Tenure 中有机会被选中作为矿工。VRF 公钥的注册状态由 globals 中的 leader_key_registration_state 成员变量维护，初始状态为 Inactive。

当节点选择成为矿工时，relayer 在处理指令的初始阶段会注册 VRF 公钥[ testnet/stacks-node/src/nakamoto_node/relayer.rs#L721](https://github.com/stacks-network/stacks-core/tree/master/testnet/stacks-node/src/nakamoto_node/relayer.rs#L721)。

1. **生成 VRF Keypair**：节点本地生成一个 VRF 密钥对（keypair）。
2. **组装注册数据**：将 VRF 公钥、BTC 最新区块的共识哈希、矿工的 Stacks 公钥等数据打包成 LeaderKeyRegister 结构，其中 vout 的索引下标需要设置为 0。
3. **提交到 BTC**：该数据结构随后被打包成一个 BTC 交易并发送到 BTC 网络，leader_key_registration_state 的状态更新为 Pending。
4. **确认状态**：当该交易在 BTC 链上得到确认时，节点会扫描新生成的 BTC 区块中所有的 key_registers 操作并比对交易 ID（txid），若匹配成功，leader_key_registration_state 状态设置为 Active。此时，节点的 VRF 公钥已成功注册，即可参与 PoX 的矿工选举。

当新的 BTC 区块被处理完毕，或新的 Tenure 出现了第一个 Stacks 区块时，选择成为矿工的节点会产生 IssueBlockCommit 类型的指令。通过[ testnet/stacks-node/src/nakamoto_node/relayer.rs#L678](https://github.com/stacks-network/stacks-core/tree/master/testnet/stacks-node/src/nakamoto_node/relayer.rs#L678) 的 issue_block_commit 函数，矿工发起出块竞标请求，参与下一个 Tenure 的出块权竞选。

在发起出块竞标请求时，矿工会使用 LeaderBlockCommitOp 结构来定义其出块承诺。以下是 LeaderBlockCommitOp 结构：

pub struct LeaderBlockCommitOp {

  pub block_header_hash: BlockHeaderHash, // hash of Stacks block header (sha512/256)

  pub new_seed: VRFSeed,   // new seed for this block

  pub parent_block_ptr: u32, // block height of the block that contains the parent block hash

  pub parent_vtxindex: u16, // offset in the parent block where the parent block hash can be found

  pub key_block_ptr: u32,  // pointer to the block that contains the leader key registration

  pub key_vtxindex: u16,  // offset in the block where the leader key can be found

  pub memo: Vec<u8>,    // extra unused byte

  /// how many burn tokens (e.g. satoshis) were committed to produce this block

  pub burn_fee: u64,

  /// the input transaction, used in mining commitment smoothing

  pub input: (Txid, u32),

  pub burn_parent_modulus: u8,

  /// the apparent sender of the transaction. note: this

  /// is *not* authenticated, and should be used only

  /// for informational purposes (e.g., log messages)

  pub apparent_sender: BurnchainSigner,

  /// PoX/Burn outputs

  pub commit_outs: Vec<PoxAddress>,

  // PoX sunset burn

  pub sunset_burn: u64,

  // common to all transactions

  pub txid: Txid,              // transaction ID

  pub vtxindex: u32,             // index in the block where this tx occurs

  pub block_height: u64,           // block height at which this tx occurs

  pub burn_header_hash: BurnchainHeaderHash, // hash of the burn chain block header

}

**sunset_burn**：用于实现 STX 增发的通缩控制机制。计算方式为：
scss
Copy code
sunset_burn = total_commit * (progress through sunset phase) / (sunset phase duration)

- 该字段会随着阶段的推进线性增加，以控制整体的供应。
- **burn_fee**：矿工为 PoX 出块权支付的燃烧费用，以 satoshi 计。PoX 的机制要求矿工支付一定的费用来增加其出块的概率。
- **commit_outs**：用于接收 PoX 燃烧费用的地址列表，通常包含两个接收地址。具体的接收地址会基于 VRF 随机选择 stackers 集合中的成员来分配https://github.com/stacks-network/stacks-core/tree/master/stackslib/src/chainstate/burn/db/sortdb.rs#L1534。

在出块竞标过程中，矿工会将 LeaderBlockCommitOp 结构的实例封装为一个 BTC 交易并发布到 BTC 网络上，随后等待该 Tenure 结束。

当当前 tenure 结束并出现新的 BTC 区块后，Stacks 系统会分析 BTC 区块中的交易，从符合 Stacks 规范的交易中选出下一个 block commit，从而确定新的矿工。

- **矿工选择**：Stacks 根据前一个选举中的 VRF 种子，以及 BTC 中矿工的 burn fee 分发情况来选择矿工。具体实现包含以下几个关键步骤：
  - 基于上次选举的 VRF seed。
  - 结合 burn fee 的分配逻辑【代码行：stackslib/src/chainstate/burn/distribution.rs#L153】。
  - 计算候选矿工的 index 值【代码行：stackslib/src/chainstate/burn/sortition.rs#L128】，从而确定获选矿工。
- **获选矿工**：根据计算得出的 index 对应的 block commit 即为新的 tenure 的矿工。这些复杂的选举逻辑涉及对 BTC 区块信息的验证以及结合特定的种子进行候选排序。

如果某节点被选中为新 tenure 的矿工，relayer 会启动一个新的矿工线程https://github.com/stacks-network/stacks-core/tree/master/testnet/stacks-node/src/nakamoto_node/relayer.rs#L591。此流程如下：

1. **打包并广播区块**：矿工线程将生成并打包区块，将该区块广播给负责的 stackers，以便他们进行签名[https://github.com/stacks-network/stacks-core/tree/master/testnet/stacks-node/src/nakamoto_node/miner.rs#L407](https://github.com/stacks-network/stacks-core/testnet/stacks-node/src/nakamoto_node/miner.rs#L407)。
2. **签名验证**：当至少 70% 的 stackers 验证并签署区块[https://github.com/stacks-network/stacks-core/tree/master/stackslib/src/net/api/postblock_proposal.rs#L192](https://github.com/stacks-network/stacks-core/stackslib/src/net/api/postblock_proposal.rs#L192)，区块会被广播至网络中[https://github.com/stacks-network/stacks-core/tree/master/testnet/stacks-node/src/nakamoto_node/miner.rs#L187](https://github.com/stacks-network/stacks-core/testnet/stacks-node/src/nakamoto_node/miner.rs#L187)。
3. **区块验证**：其他节点接收到新区块后，会进行合法性验证[https://github.com/stacks-network/stacks-core/tree/master/stackslib/src/chainstate/nakamoto/mod.rs#L1691](https://github.com/stacks-network/stacks-core/testnet/stacks-node/src/nakamoto_node/miner.rs#L187)。

- **第一个区块**：如果是 tenure 的第一个区块，该区块中必须包含 tenurechange 类型的交易和 coinbase 类型的交易[https://github.com/stacks-network/stacks-core/tree/master/testnet/stacks-node/src/nakamoto_node/miner.rs#L187](https://github.com/stacks-network/stacks-core/testnet/stacks-node/src/nakamoto_node/miner.rs#L187)。
- **tenure extend** **类型区块**：此类区块仅需验证是否包含 tenurechange 类型交易，不需要 coinbase 类型交易，同时需要验证是否与上一个 tenure 的共识哈希一致【代码行：stackslib/src/chainstate/nakamoto/mod.rs#L678】。

除此之外，还需对区块中的矿工身份和交易格式进行额外检查。

当所有检查通过后，区块将被存储到数据库中，同时更新链上状态。

如果一个新的 BTC 区块出现后没有其他矿工获得出块机会，理论上前 tenure 的矿工将继续打包区块，但在目前的实现中，上一 tenure 的矿工会直接结束 tenure [https://github.com/stacks-network/stacks-core/tree/master/testnet/stacks-node/src/nakamoto_node/relayer.rs#L652](https://github.com/stacks-network/stacks-core/testnet/stacks-node/src/nakamoto_node/miner.rs#L187)

若在新的 tenure 中该矿工未赢得出块权，relayer 会终止该矿工线程来结束 tenure。结束流程不需要额外的 BTC 交易或共识步骤，仅是终止矿工线程即可[https://github.com/stacks-network/stacks-core/tree/master/testnet/stacks-node/src/nakamoto_node/relayer.rs#L608。](https://github.com/stacks-network/stacks-core/testnet/stacks-node/src/nakamoto_node/miner.rs#L187)

问题：如果block commit交易没有如期上链，如何处理？

stackslib/src/chainstate/burn/operations/leader_block_commit.rs#L992 会将其列入missed commit中。

# Stacks是如何与BTC通信的

Stacks 使用 BTC 区块链的 OP_RETURN 操作来打包并存储需要的数据，并将其通过 vout 形式的交易输出发送至 BTC 网络。相关代码位于：[testnet/stacks-node/src/burnchains/bitcoin_regtest_controller.rs#L2129](https://github.com/stacks-network/stacks-core/tree/master/testnet/stacks-node/src/burnchains/bitcoin_regtest_controller.rs#L2129)。不同类型的 Stacks 操作会生成不同的 OP_RETURN 数据，具体包括：

以下是 Stacks 系统支持的主要操作类型：[stackslib/src/chainstate/burn/operations/mod.rs#L337](https://github.com/stacks-network/stacks-core/tree/master/stackslib/src/chainstate/burn/operations/mod.rs#L337)：

- **LeaderKeyRegister**：用于注册 VRF 公钥，使矿工具备参与下一个 tenure 选举的资格。
- **LeaderBlockCommit**：用于矿工提交出块竞选请求，争取下一个 tenure 的出块权。
- **PreStx**：该类型的作用尚不明确，可能与早期的测试操作有关。
- **StackStx**：用于将 STX 进行 stacking 操作，以支持 PoX（Proof of Transfer）共识。
- **TransferStx**：用于 STX 代币的普通转账操作。
- **DelegateStx**：允许用户将 STX 委托给其他地址进行 stacking。虽然 DelegateStx 操作结构中并没有直接指定目的地址，但可能是通过合约中映射实现地址的委托关系。
- **VoteForAggregateKey**：用于为特定矿工地址投票，提升该矿工在下一次选举中的权重，从而增加其获选概率。

Stacks 的节点通过 indexer 模块同步 BTC 区块链状态，其中 indexer 位于：[testnet/stacks-node/src/burnchains/bitcoin_regtest_controller.rs#L275](https://github.com/stacks-network/stacks-core/tree/master/testnet/stacks-node/src/burnchains/bitcoin_regtest_controller.rs#L275)，而非位于 coord 模块。其运行流程如下：

1. **区块下载**：indexer 下载最新的 BTC 区块，并将这些区块信息传递给 coordinator，用于进一步的状态同步和处理。
2. **通信机制**：indexer 与 coordinator_channel 通信，以确保区块处理的消息传递顺畅。在同步每个 BTC 区块时，Stacks 会将同步的区块消息通过 channel[ stackslib/src/burnchains/burnchain.rs#L1614](https://github.com/stacks-network/stacks-core/tree/master/stackslib/src/burnchains/burnchain.rs#L1614) 发送至 coordinator，触发相应的区块处理流程[ stackslib/src/chainstate/nakamoto/coordinator/mod.rs#L540](https://github.com/stacks-network/stacks-core/tree/master/stackslib/src/chainstate/nakamoto/coordinator/mod.rs#L540)。
3. **区块消息处理**：通过上述 channel 机制，coordinator 可以高效地接收到 BTC 区块的变化，并根据其中包含的 Stacks 操作进行解析和状态更新。

# 矿工是如何出块的

出块的主要逻辑位于矿工的 mine_block 函数中，具体实现可以参考[ testnet/stacks-node/src/nakamoto_node/miner.rs#L738](https://github.com/stacks-network/stacks-core/tree/master/testnet/stacks-node/src/nakamoto_node/miner.rs#L738)。

在矿工开始构建区块时，首先会检查当前区块是否是 **tenure**（任期）的第一个区块。如果是第一个区块，矿工会将 **tenure change** 和 **coinbase** 类型的交易添加到区块头部，这两类交易分别表示矿工的身份变化和奖励的分配。具体代码位置可以参考[ stackslib/src/chainstate/nakamoto/miner.rs#L441](https://github.com/stacks-network/stacks-core/tree/master/stackslib/src/chainstate/nakamoto/miner.rs#L441)。

除去 **tenure change** 和 **coinbase** 类型的交易，矿工还会根据当前周期的 **signers** 信息，从链上的质押合约中读取相关数据，并将 **signer transactions** 加入到区块中。这些 **signer txs** 是为了后续区块验证时使用，确保区块的合法性。Stacks 网络不允许出空块，因此在矿工打包区块时，必须确保签名者的信息已经加入到区块中。

在处理交易之前，矿工还需要进行一些 **区块初始状态的初始化**。这包括：

- 初始化 **Clarity VM** 的状态，以确保合约执行的环境处于正确的初始状态。
- 分配矿工奖励和计算当前奖励周期内的奖励。
- 解锁一定数量的 token，确保交易可以顺利执行。
- 处理从 BTC 网络发出的与 STX 相关的操作，例如 STX 转账等。

这些初始化操作的结果会以 **transaction receipt**（交易回执）的形式被保存。相关代码可见[ stackslib/src/chainstate/nakamoto/mod.rs#L2492](https://github.com/stacks-network/stacks-core/tree/master/stackslib/src/chainstate/nakamoto/mod.rs#L2492)。

在初始化工作完成后，矿工才会开始从 **mempool** 中筛选可打包的交易。相关逻辑可以在[ stackslib/src/core/mempool.rs#L1601](https://github.com/stacks-network/stacks-core/tree/master/stackslib/src/core/mempool.rs#L1601) 找到。

在进行 mempool 遍历时，矿工会首先从 **cache** 中抓取交易，cache 中存储的是块内重试的交易。如果发现某个交易的 **nonce** 大于当前区块中存储账户状态的交易，矿工会认为该交易可能有前置交易，进而将其存放到 cache 中待后续处理。

接下来，矿工会按照交易的 **fee rate**（费用率）从 mempool 中选择交易。对于 **fee rate 为 NULL** 的交易，矿工会根据预设的规则进行处理，而对于有 **fee rate** 的交易，矿工会依据 **fee rate** 来决定交易的优先级。

在打包交易前，矿工需要对交易进行类型检查，排除不需要的交易类型或不考虑的交易发起者。例如，某些特定类型的交易可能会被忽略或过滤掉，以保证区块的有效性。

在交易打包过程中，矿工还需要确保区块大小不超过三个限制：

- **合约创建大小限制**：限制包含合约创建交易的大小。
- **区块燃料费限制**：限制区块内的燃料费消耗。
- **区块大小限制**：限制区块的总大小。

这些限制确保了区块的大小在合理范围内，避免因为区块过大导致网络拥堵。

在执行交易之前，Stacks 会对每笔交易进行静态检查，尤其是对部署合约类型的交易。静态检查包括对交易的语法和结构进行验证，确保合约能够在预期的环境中正确执行。静态检查的代码可以参考[ stackslib/src/net/relay.rs#L1351](https://github.com/stacks-network/stacks-core/tree/master/stackslib/src/net/relay.rs#L1351)。

经过前述的一系列检查后，矿工才会执行交易。交易执行的代码位于[ stackslib/src/chainstate/stacks/db/transactions.rs#L1461](https://github.com/stacks-network/stacks-core/tree/master/stackslib/src/chainstate/stacks/db/transactions.rs#L1461)。

在执行过程中，Stacks 网络会确保所有交易的逻辑被正确执行，并更新相关的账本状态，最后将有效的交易纳入区块中。

在 Stacks 中，一共有六种主要的交易类型，分别是：

1. **TokenTransfer**：转账交易，用于将 STX 或其他代币从一个账户转移到另一个账户。
2. **ContractCall**：合约调用交易，允许调用已经部署的智能合约中的函数。
3. **SmartContract**：部署合约交易，用于将新的智能合约部署到 Stacks 网络。
4. **PoisonMicroblock**：类似于 Massa 中的 **Denunciation**，用于证明某个矿工的不诚实行为，即举报矿工的欺诈行为。
5. **Coinbase**：该交易类型仅在 **tenure**（任期）开始时的第一个区块中存在，用于分配矿工奖励。
6. **TenureChange**：该交易类型与 **Coinbase** 类似，主要用于共识和证明当前矿工身份的变更。

所有的交易都会在 **Clarity VM** 的上下文中执行。Clarity 是 Stacks 网络中执行智能合约的虚拟机，用于对交易进行验证并执行合约的逻辑。具体代码参考[ stackslib/src/chainstate/stacks/db/transactions.rs#L961](https://github.com/stacks-network/stacks-core/tree/master/stackslib/src/chainstate/stacks/db/transactions.rs#L961)。

对于合约调用类型的交易，Stacks 允许设置 **postcondition**，用来检查交易执行后是否满足预期条件。这个 **postcondition** 主要用于验证合约资产（如某些代币）的增减是否符合预期。例如，检查某个特定的代币是否在调用合约后增加或减少了预定数量。相关实现可见[ stackslib/src/chainstate/stacks/db/transactions.rs#L566](https://github.com/stacks-network/stacks-core/tree/master/stackslib/src/chainstate/stacks/db/transactions.rs#L566)。

交易执行后的结果会通过 **Receipt** 结构进行保存。这个结构用于保存交易执行的结果，包括是否成功执行、所更改的状态等信息。相关实现位于[ stackslib/src/chainstate/stacks/events.rs#L47](https://github.com/stacks-network/stacks-core/tree/master/stackslib/src/chainstate/stacks/events.rs#L47)。

在所有交易执行并验证后，矿工会构建新区块。区块的头部会根据当前的交易结果计算生成，相关代码在[ stackslib/src/chainstate/nakamoto/miner.rs#L345](https://github.com/stacks-network/stacks-core/tree/master/stackslib/src/chainstate/nakamoto/miner.rs#L345) 中定义。

随后，矿工会对区块进行签名。区块签名的目的是确保该区块的有效性，并确认矿工对该区块的创建和内容负责。签名完成后，矿工会将区块广播到网络中。

当广播出去的区块被接收到时，网络中的其他节点会对该区块进行验证。验证主要包括以下几个方面：

- **区块格式**：确保区块的结构和格式符合协议要求。
- **BTC 上的 PoX 信息**：验证区块内的 PoX 相关信息是否与当前的 BTC 链的状态一致。
- **区块内部的一致性**：检查区块内交易的一致性，确保没有恶意篡改或错误的交易。
- **Stackers 的验证签名**：验证区块中的 Stackers 签名，确保矿工获得足够的支持。

这部分的代码实现位于[ stackslib/src/chainstate/nakamoto/mod.rs#L1691](https://github.com/stacks-network/stacks-core/tree/master/stackslib/src/chainstate/nakamoto/mod.rs#L1691)。如果所有检查都通过，区块会被存储到 **staging_db** 中。

广播到 P2P 网络的区块会被接收到并通过 **relayer** 的 process_network_result 函数进行处理。具体实现参考[ stackslib/src/net/relay.rs#L2114](https://github.com/stacks-network/stacks-core/tree/master/stackslib/src/net/relay.rs#L2114)。

由于该区块已经被多个 Stackers 验证，新的区块仅需要进行简单的 **AST**（抽象语法树）检查和签名验证。然后，矿工会通知 **coordinator** 执行该区块的相关逻辑。

在 **coordinator** 执行区块时，会进行一系列额外的检查，如验证区块中的 **VRF**（验证随机函数）等信息。执行过程的逻辑和矿工的出块逻辑大致一致，具体实现可以参考[ stackslib/src/chainstate/nakamoto/mod.rs#L2715](https://github.com/stacks-network/stacks-core/tree/master/stackslib/src/chainstate/nakamoto/mod.rs#L2715)。

# STACKING

# Clarity

# SECURITY

### **Consensus Mechanism Security**

**General:**

- **Reliance on Bitcoin for Security:** Stacks’ security is directly tied to Bitcoin. Stacks’ state is recorded on the Bitcoin blockchain, making the cost of attacking Stacks blocks inherently linked to the cost of attacking Bitcoin blocks.
- **Decentralization:** The tasks of block production and consensus are assigned to different roles, with miners and stackers being selected randomly through the VRF mechanism, ensuring decentralization in the process.
- **Selfish Mining:** The random selection of miners helps mitigate the problem of selfish mining by Bitcoin miners, reducing the potential for collusion and optimizing overall network security.

**Miner Security:**

- **Dynamic Balance in Competitive Conditions:** In order to increase their chances of producing a block, miners are incentivized to increase the PoX cost through the random selection algorithm, while their potential rewards are unknown until the tenure begins.
- **MEV (Miner Extractable Value):** Since miner competition does not occur during the block production phase, there is no risk of blockchain reorganization (reorg) in Stacks. This prevents other miners from attempting to reorganize Stacks blocks for MEV profits.
- **Block Validation:** To ensure that miners behave as expected during their tenure, Stacks performs validation voting on each block before it is added to the chain. A block is only added if 70% of the stackers responsible for that tenure validate and sign it.

**Ecosystem:**

- **Stacking:** Stakers are required to lock their STX in the staking contract, ensuring that Stacks’ token economy remains secure and stable.
- **Rewards:** Stakers earn BTC rewards from their STX staking, while miners spend BTC to produce blocks and receive STX as their reward.

### **Smart Contract Security**

- **Clarity Language:** 
  - 解释性语言
  - 不允许重入
- **Static Analysis and Determinism:** 
  - 静态分析
  - 确定性

### **Data Integrity and State Consistency**

- **Bootstrap and Sync Mechanism:**
  - **Compatibility**: Stacks has undergone multiple years of version upgrades, with different blockchain code being used across epochs. However, Stacks ensures forward compatibility when updating its code, maintaining the integrity of the system over time.
  - **L1-driven Block State Updates**: All actions occurring within Stacks are based on updates from Bitcoin's (L1) block state. This means that Stacks relies on Bitcoin's consensus and block state to drive the synchronization and state transitions within the Stacks network, ensuring that the system remains aligned with Bitcoin.

- **Handling Forks and Reorganizations:** 	
  - todo
- **Deterministic State and NaN Unification:**
  - todo

### **Potential Attack Vectors**

- **Todo**

# BTC与L2交互可能存在的安全问题以及Stacks做法

## 写入

#### **BTC写入数据限制**

比特币的 OP_RETURN 字段仅能存储 80 字节，L2 的写入内容需要高度优化。过多的数据写入不仅占用比特币网络资源，还会显著增加费用成本。

- **需要注意的地方**
  - 确定哪些数据是写入比特币链的必要内容，减少冗余数据。
  - 数据应该为L2提供必要的验证信息与可追溯数据，为 L2 的安全提供支撑，使得在。
- **Stacks 的处理方式**
  - 仅写入与 PoX 共识、区块提交和矿工奖励分配相关的核心数据（如 LeaderBlockCommit）。
  - 压缩和摘要化数据，仅保留必要的哈希、地址等关键数据摘要，并且避免将数据分散到多个 OP_RETURN。

#### **数据可靠性**

BTC L2的去中心化设计导致写入数据的有效性在发送到BTC之前未被所有矿工和节点严格验证，BTC上存储的数据可能导致共识分歧或攻击者注入恶意数据。

- **需要注意的地方**
  - 确保数据包含可以验证的字段，避免无效或错误数据进入比特币区块后破坏共识或L2区块数据。
- **Stacks 的处理方式**
  - 矿工在提交与共识相关stacks操作类型的BTC交易包含内容验证数据（如VRF、共识哈希等）并与当前区块链状态匹配。

#### **交易延时**

BTC L2向BTC发送的数据可能并不能及时的在目标的区块上链，由于网络拥堵或者其他原因可能导致交易会延时几个区块才被BTC确认。

- **需要注意的地方**
  - 确保部分未能如期被BTC确认的交易不会影响L2的整体运作，关键的共识和确认逻辑应该独立于BTC。
- **Stacks 的处理方式**
  - 对于过期LeaderBlockCommit交易，stacks会将发送给stackers的BTC算作增发的STX的一部分。
  - BTC上只存储POX的证明，BTC上的数据可以用作计算POX过程的数据，而不是结果，所以部分交易的过期不会导致POX的失败。

## 读取

#### **比特币链状态的延迟获取**

比特币的链上状态更新速度较慢，L2 的读取频率可能不匹配实际需求，并且BTC出块时间并不是严格固定。

- **需要注意的地方**
  - L2 必须实时感知比特币区块的变化，尤其是与自身共识或交易验证相关的状态，如共识与同步数据。
  - L2的状态更新需要以BTC的状态更新为依据，BTC区块的更新决定L2的更新。
- **Stacks 的处理方式**
  - 通过 indexer 模块，从比特币节点监听区块和交易更新，确保获取最新的链上状态，并且以BTC区块的更新为stacks区块和网络更新的驱动。
  - 使用轻量级的数据筛选机制，仅解析与 Stacks 相关的比特币数据（如 OP_RETURN 数据）。

#### **比特币重组与分叉**

比特币可能发生区块链重组，L2 的读取状态可能基于一个非最终的比特币区块，导致错误决策或回滚需求。

- **需要注意的地方**
  - 检测比特币链的分叉或重组，确保使用的是主链上的状态。
  - 在链重组发生时，能安全回滚到正确的状态。
- **Stacks 的处理方式**
  - todo（stacks的做法是由多个stackers来监听BTC区块链，当BTC发生分叉或重组时，stackers会在不同的分支上分别任命某一个矿工作为该tenure的矿工。由于矿工出块需要获得stackers的确认，只有认为该块符合某个stacker观测的分支时，该stacker才会签名。即便发生分叉或重组，也会保证至少有一个分支会在BTC的主分叉上记录，在非主要分支上的块，则需要进行重新的同步。

对于一些极端情况，如大多数stacker都没有监听到正确的BTC主分支或者重组的情况，stacks目前还没有解决。）

#### **比特币交易处理**

比特币可能会发起与L2的通信，这些通信会对链上状态进行更改。

- **需要注意的地方**
  - 确保对L2的状态的写入和更改在正确的区块位置进行处理，并且保持一致。
  - 比特币发往L2的消息可能需要信息不够完整，比如某些类型的数据需要超过80字节表示，需要利用L2的一些特性。
- **Stacks 的处理方式**
  - 所有的来自BTC的交易都被写入tenure开始区块的头部位置。
  - 利用stacks的合约存储地址关系映射，使得BTC交易可以充分使用80字节的空间。

## 同步

#### **不同步或丢失数据**

比特币网络上的数据同步可能由于网络延迟、节点配置问题或区块广播失败导致不一致。L2 节点可能基于过时或错误的比特币状态执行操作。

- **需要注意的地方**
  - L2 节点需要具有完整、及时的比特币区块和交易信息。
  - 设计去中心化的同步机制，避免因单点失效影响网络运行。
- **Stacks 的处理方式**
  - 通过比特币的多个全节点监听网络，确保数据的完整性和可靠性。
  - 在同步比特币数据时，将其结果发送到 Channel 中，由 Coordinator 统一协调处理，避免分散逻辑带来的风险。

#### **不同版本的兼容性**

L2 系统可能在长期运行中经历多次协议升级，而比特币的状态解析逻辑需要兼容不同的版本和规则。

- **需要注意的地方**
  - 确保升级不会破坏现有节点的同步能力或引发数据验证错误。
- **Stacks 的处理方式**
  - 实现多 Epoch 的向后兼容机制，在区块链升级时，确保旧版协议解析逻辑仍能适用。
  - 在区块头中添加版本信息，使节点能够识别并切换到对应的解析逻辑。