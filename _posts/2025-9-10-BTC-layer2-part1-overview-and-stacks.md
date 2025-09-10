# BTC Layer2

一年多前写的老东西，为了整理成文档翻译成了英文，懒得翻译回去了，仅作记录。

## What is Layer2

The current definition of L2 is too broad; anything that interacts with the BTC chain to read or write its state can be called L2, such as inscriptions or staking.

However, I do not agree with this definition. In general, these broad definitions make it very difficult for people who are new to the L2 space. Similar projects may label themselves as L2 as a gimmick to attract investors, and retail investors on social media do not delve deeper, simply promoting any project as L2.

In my understanding, L2 must be designed to expand the functionality of L1, such as through smart contracts, or to improve efficiency, like TPS, or to reduce costs, such as transaction fees. Only such projects should be called L2.

When discussing L2 projects, there are often different definitions or terms to describe what kind of "L2" a project is. The most mainstream descriptions for similar projects are divided into: state channels, sidechains, rollups, and Layer 2. These are usually classified under the broad category of Layer 2, yet when it comes to describing the logic of a particular project, it's often unclear how to distinguish these concepts, which can be confusing.

Here, based on some collected materials and personal understanding, I offer a rough distinction between these three concepts.

Regardless of the category, from BTC's perspective, these projects can be seen as a black box. BTC sends a transaction to an interaction point with one of these projects, and after some time, this black box returns the result of the execution.

### state channel

State channels are a solution for users who want to perform complex operations, such as futures and contracts, that may be difficult or very complex to implement directly on Bitcoin. The main idea behind state channels is to offload these complex computations to off-chain processes, with only the final results being recorded on-chain. The execution process typically does not involve stringent verification and consensus mechanisms; it focuses on basic execution without addressing other aspects.

The most typical project in this category is the **Lightning Network**.

### side chain

Sidechains enhance the functionality of state channels by implementing an independent blockchain, with the key difference being that sidechains address the consensus issue, which state channels might overlook, potentially introducing security concerns. Sidechains operate as independent chains with their own native currency. They are called sidechains because they communicate with the main chain (L1), typically for asset transfers, similar to the function of a cross-chain bridge. These bridges are often an inherent feature of the sidechain project. In summary, the main characteristic of a sidechain is that it is an independent blockchain with a native cross-chain bridge.

A typical project in this category is **Rootstock (RSK)**.

### roll-up

The concept of roll-up is vague; it generally refers to the process of bundling a set of data and sending it to the blockchain. Roll-up data typically consists of results from computations performed off-chain from Layer 1 (L1), which may be used for verification or as a scaling solution, aggregating multiple transactions into one package. This data often stores part or all of its information within L1 blocks, effectively using L1 as a trusted and public data storage facility.

Common types of roll-ups include zk-rollups and optimistic rollups, with the main difference lying in the method and content of the data being packaged.

### Layer2

Layer 2 can be understood as a combination of sidechains and rollups. Layer 2 typically implements a complete blockchain, but its consensus mechanism and native currency may be deeply integrated with Layer 1 (L1). Unlike sidechains, which have a consensus mechanism entirely independent of L1, Layer 2 writes block information to L1, meaning that to execute a 51% attack, an attacker would need to control both L2 and L1 blocks. The myriad of Layer 2 projects on the market often focus on innovations in rollup techniques or consensus methods, either to genuinely solve certain issues or as a marketing strategy to attract users and investors.

A typical project in this category is **Stacks**

## Differences Between BTC L2 and ETH L2

### How L2 Works

When discussing Layer 2 (L2) solutions, we often refer to two main rollup strategies: Optimistic Rollups and Zero-Knowledge (ZK) Rollups. These strategies are currently the leading L2 rollup solutions. To understand the differences between BTC L2 and ETH L2, let’s first examine how ETH L2 operates.

Optimism’s approach involves a sequencer responsible for packaging L2 blocks and periodically writing the packaged L2 state information to the L1 rollup contract. This information includes block headers, block states, and account states' Merkle roots. Due to the centralized nature of the sequencer role, Optimism introduces a challenge mechanism to prevent issues arising from sequencer malfeasance or downtime. The sequencer must send all L2 information—every block's content—to L1. Since this detailed data is significantly larger than the compressed Merkle root, it is usually sent to the L1 rollup contract address as calldata or in blob form. This Data Availability (DA) method ensures that if the sequencer fails or acts maliciously, other users can either reconstruct the entire L2 state based solely on L1 or generate a fraud proof based on the L2 state recorded on L1, thus ensuring L2 security.

ZK Rollups take a slightly different approach. Instead of using a challenge mechanism, ZK Rollups generate Zero-Knowledge proofs to verify that the sequencer's rollup results are correct. If the proof is valid and passes verification by the L1 contract, the result is considered correct.

In discussing the overarching concept of L2, it’s clear from these two dominant rollup strategies and the broader vision of L2 that the core idea is to reduce decentralization by eliminating the consensus process. Instead, L2 relies on challenge mechanisms or ZK proofs to ensure the centralized outcomes are correct, thereby improving throughput. At the same time, by writing the entire blockchain data to L1 (Data Availability), the instability risks associated with centralization are mitigated.

### What BTC Can and Cannot Do

Compared to Ethereum, Bitcoin's functionality is limited. According to the official Bitcoin documentation ([https://developer.bitcoin.org/devguide/transactions.html](https://developer.bitcoin.org/devguide/transactions.html)), the transactions supported by Bitcoin are confined to operations such as transfers and signatures. Although many projects are striving to unlock the potential of Bitcoin’s scripting language, due to Bitcoin’s UTXO model, which does not support state read/write operations, and the lack of Turing completeness in Bitcoin scripts (no support for arbitrary branching or loops), it is impossible to implement contracts like Uniswap on Ethereum using Bitcoin’s current architecture.

> Bitcoin smart contracts, similar to those on other networks, are code snippets that automatically carry out specified actions when certain conditions are met. It uses its own programming language, Script, which operates on a "lock and key" system to execute smart contracts. When initiating a transaction, the sender establishes a condition or rule acting as the "lock." At the same time, the recipient must provide a corresponding "key" in the form of code in Script to fulfill the sender's condition.

Bitcoin's scripting language can handle simple scenarios, such as transfers and multi-signature transactions. As the official Bitcoin documentation notes, Bitcoin’s script functions more like a lock and key—where the transaction initiator defines a lock, and only those with the correct key can spend the funds.(https://en.bitcoin.it/wiki/Contract)

Additionally, Bitcoin’s data throughput is much lower compared to Ethereum. Bitcoin’s block size ranges from 1 MB to 1.4 MB (with SegWit), while Ethereum's block size varies with gas usage, ranging from a few hundred KB to 2.5 MB (post-EIP-4844). Furthermore, Ethereum’s block time is significantly faster, with block production occurring at a rate several times that of Bitcoin. In terms of speed, Bitcoin is clearly at a disadvantage.

### Characteristics and Challenges of BTC L2

**Conclusion:** BTC Layer 2 (L2) solutions are inherently decentralized, requiring their own consensus mechanisms. They also have a greater need to issue new tokens compared to ETH L2 and are more sensitive to fluctuations in the network state of Layer 1 (L1), such as reorgs and forks. In comparison to ETH L2, BTC L2 functions more like a sidechain.

1. **Higher Demand for Issuing New Tokens:** In some L2 projects, the L2 does not issue its own tokens but uses a cross-chain bridge mechanism to implement deposit logic on L1 through contracts, with the native L1 token serving as the L2's native token. However, since Bitcoin does not support such contract implementations, achieving a similar cross-chain bridge functionality on BTC would require a centralized account or multi-signature setup to perform cross-chain operations.
2. **Requirement to Implement Its Own Consensus Mechanism:** ETH L2 relies on challenge mechanisms and ZK proofs to address security and trust issues, with the final implementation depending on the L1 rollup contract. Since BTC's script cannot execute such complex contracts, similar mechanisms cannot be implemented on BTC. Consequently, BTC L2 cannot adopt the centralized recording approach used by ETH L2. To resolve issues of centralization and trust, BTC L2 must establish its own consensus mechanism.
3. **Limited Reliance on BTC as Data Availability (DA):** Due to BTC's limited block size and the small capacity (80 bytes) of the opreturn opcode, BTC L2 can only store minimal information, such as block hashes, on the BTC blockchain. The detailed state of the block cannot be fully written to BTC.
4. **Slower Speed Compared to ETH L2:** BTC's longer block time results in slower transaction confirmation. Additionally, BTC L2’s higher degree of decentralization, requiring consensus validation, further extends block production time compared to the more centralized approach of ETH L2.
5. **Greater Cost in Handling L1 Reorgs:** When Ethereum experiences a reorg, it typically affects only a small number of sequencer block reorganizations. However, in the case of a BTC L2, all nodes in the network must participate in consensus confirmation, leading to higher costs and greater impact.

## Stacks

https://docs.stacks.co/

https://github.com/stacks-network/stacks-core

### PoX

[PoX](https://github.com/stacksgov/sips/blob/main/sips/sip-007/sip-007-stacking-consensus.md) (Proof of Transfer) is the consensus algorithm used by Stacks, and it evolved from the [PoB](https://github.com/stacksgov/sips/blob/main/sips/sip-001/sip-001-burn-election.md) (Proof of Burn) consensus algorithm.

The underlying design logic of consensus algorithms is generally similar: they aim to verify the correctness of block content, proving the honesty of nodes. By introducing costs for proof and penalties for malicious behavior, the intention is to deter dishonest nodes from disrupting the network. Incentive mechanisms encourage more nodes to participate and maintain the integrity of the blockchain, making it extremely costly for a single malicious node to act against the network.

Both PoB and PoX require a trusted and robust L1 ecosystem for support. The core idea behind both is that miners spend a certain amount of an asset to prove their honesty, thereby earning the opportunity to produce blocks. In the Stacks ecosystem, Bitcoin (BTC) serves as the L1.

In the PoX consensus mechanism, there are two roles: miners and stackers.

Miners, similar to Ethereum’s builders, are responsible for packaging blocks. To increase the chances of their block being added to the chain, miners transfer a certain amount of Bitcoin to a specific address—the more they transfer, the higher the chances their block will be included.

In Stacks, each block corresponds to a Bitcoin block. To improve speed, Stacks divides each block into multiple microblocks. Miners package these microblocks into a single block for final consensus validation.

Stackers, akin to validators in Ethereum, are required to stake STX (the native token of Stacks). These stakers, known as stackers, use VRF (Verifiable Random Function) to select which miner’s block will be added to the chain during each block cycle.

Once the block is added to the chain, miners receive STX and transaction fees as rewards, while stackers earn a portion of the BTC spent by miners in PoX and a share of the STX.

**Q: Where is BTC transferred to? What happens to the miner’s spending if the block isn’t added to the chain?**

**A: BTC transfered to a randomly selected address among stackers, the higher stacker stake their STX, the more likely they be selected to be that address, if a miner failed to be current block's miner, they wont get refund.**

### Clarity

Clarity is a language specifically designed for Stacks. It is similar to lower-level languages like Vyper or Huff. You can see an example in the [official staking contract](https://explorer.hiro.so/txid/SP000000000000000000002Q6VF78.pox-4?chain=mainnet).

Compared to Solidity, Clarity has five main characteristics:

1. **Interpretation-based language** (no need for compilation).
2. **No recursive calls allowed** (preventing reentrancy attacks).
3. **User-defined post-conditions** can be set in transactions (transaction fails if conditions are not met).
4. **Mandatory return value handling**.
5. **Ability to query BTC state**.

### Nakamoto Upgrade

The Stacks documentation often references an important update called the Nakamoto Upgrade, aimed at resolving several legacy issues within the Stacks ecosystem, including:

1. Slow network speed due to Bitcoin's block time, fork issues, and the lack of block ordering mechanisms leading to disorder in on-chain applications.
2. The microblocks mechanism failing to significantly improve transaction processing speed.
3. Stacks forks not being bound to BTC, resulting in low reorder costs.
4. Weak connections between miners, leading to multiple forks in Stacks.
5. Some Bitcoin miners also running Stacks, which introduces risks of centralization and MEV (Miner Extractable Value) as malicious miners might deliberately prevent other miners' commits from being added to the chain.

The Nakamoto Upgrade introduces several key changes to address these problems:

1. **New Block Production Method:** The previous microblock design is replaced with the concept of "tenure." The idea is that block production time remains constant, with a fixed number of Stacks blocks produced within each Bitcoin block time. Miners are responsible for producing blocks during a specific tenure, with PoX consensus and stackers determining which miner produces blocks during that period.
2. **Stackers' Deeper Involvement in Block Production:** Stackers now play a more active role in validating, storing, broadcasting, and signing each block.
3. **Stronger Miner Coordination:** Previously, Stacks blocks only required the recording of block hashes on Bitcoin, leading to numerous forks. Now, miners must submit the hash of the first block from the previous tenure as an `indexhash` parameter when producing blocks, strengthening the connection between different miners.
4. **Transaction Confirmation Requires Two Bitcoin Block Times:**

Let’s break down the life cycle of a transaction in Stacks after this upgrade:

1. Alice creates a transaction and sends it to node Bob.
2. Bob verifies the transaction, broadcasts it, and adds it to the mempool.
3. Charlie, the current tenure miner, includes Alice's transaction in a block.
4. Charlie sends the packaged block to signers for validation (signers can be understood as a module of Stacks nodes, with the role of signers being played by stackers).
5. The block is accepted or rejected by signers. If over 70% of signers accept the block, it is added to the chain.
6. This process repeats throughout Charlie's tenure.
7. At the end of Charlie’s tenure, if Dave is the next miner, he will:

   1. Participate in PoX and submit a block commit transaction to the Bitcoin network, which includes an `index block hash` field that refers to the hash of the first block in Charlie's tenure.
   2. Stackers use PoX and VRF to select Dave as the next tenure miner, signing a tenure change transaction on Stacks that specifies block N as the last block produced by Charlie and mandates that Dave must build on block N (no forking allowed).
   3. Dave receives the tenure change transaction, starts producing blocks, and includes the tenure change transaction as the first transaction in his block.
8. Alice’s transaction is finalized only after Dave’s tenure is complete, which takes at least two Bitcoin block times.
