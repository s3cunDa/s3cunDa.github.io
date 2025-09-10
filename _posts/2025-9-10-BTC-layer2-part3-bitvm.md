BTC L2系列的第三篇，关于BitVM

原版用中文写的，后来整理成英文文档原版找不到了，凑合看吧。

Research Document on BitVM and BitVM Bridge: Technical Principles and Examples

# Introduction

This document aims to provide a comprehensive technical overview of BitVM and BitVM Bridge, exploring their underlying principles and illustrating their functionalities through practical examples. We will delve into the intricacies of each concept, highlighting their significance in the broader blockchain ecosystem.

# Background

## BTC

Bitcoin (BTC) utilizes a model called the "Unspent Transaction Output" (UTXO) to manage assets. Its core architecture can be abstractly understood as a "lock and key" mechanism: Each UTXO is associated with a cryptographic lock (typically defined by a script), and only a user possessing the correct "key" (usually referring to signatures or data that satisfy the script's conditions) can unlock and spend it.

Unlike blockchain platforms such as Ethereum (ETH), which support native smart contracts, Bitcoin inherently lacks native smart contract capabilities. This stems primarily from two key technical limitations:

1. Absence of State Storage: BTC's UTXO model is inherently stateless. Each transaction solely consumes old UTXOs and creates new ones; the system lacks a global or persistent state storage for smart contracts to read or modify intermediate state during execution. State storage is fundamental for running complex, interactive smart contracts.
2. Limited Script Functionality (Non-Turing Complete): BTC's transaction scripting language (Script) is deliberately designed to be non-Turing complete. It does not support looping constructs or complex conditional jumps, thereby preventing the possibility of infinite loops and enhancing network security and predictability. However, this also means it cannot theoretically execute arbitrary complex computational logic, which is a core characteristic of Turing-complete languages (like Ethereum's Solidity) and is essential for implementing general-purpose smart contracts.

## L2

The current definition of L2 is too broad; anything that interacts with the BTC chain to read or write its state can be called L2, such as inscriptions or staking.

However, I do not agree with this definition. In general, these broad definitions make it very difficult for people who are new to the L2 space. Similar projects may label themselves as L2 as a gimmick to attract investors, and retail investors on social media do not delve deeper, simply promoting any project as L2.

In my understanding, L2 must be designed to expand the functionality of L1, such as through smart contracts, or to improve efficiency, like TPS, or to reduce costs, such as transaction fees. Only such projects should be called L2.

When discussing L2 projects, there are often different definitions or terms to describe what kind of "L2" a project is. The most mainstream descriptions for similar projects are divided into: state channels, sidechains, rollups, and Layer 2. These are usually classified under the broad category of Layer 2, yet when it comes to describing the logic of a particular project, it's often unclear how to distinguish these concepts, which can be confusing.

Here, based on some collected materials and personal understanding, I offer a rough distinction between these three concepts.

Regardless of the category, from BTC's perspective, these projects can be seen as a black box. BTC sends a transaction to an interaction point with one of these projects, and after some time, this black box returns the result of the execution.

# BitVM

## Technical Principles

BitVM is a novel concept that enables the execution of complex computations on Bitcoin without altering its core protocol. It achieves this by leveraging Bitcoin's existing scripting capabilities to create a verifiable computation model. The core idea revolves around transforming any arbitrary computation into a series of simple, verifiable steps that can be executed and checked on the Bitcoin blockchain. This process involves:

- **Proof Generation:** A prover generates a proof for a given computation off-chain.
- **Challenge-Response Protocol:** A verifier can challenge the prover if they suspect the proof is invalid. This initiates a dispute resolution mechanism on the Bitcoin blockchain.
- **Fraud Proofs:** If the prover is indeed malicious, the verifier can submit a fraud proof to the Bitcoin network, revealing the invalidity of the computation.

## BitVM Design Objectives and Technical Architecture

BitVM aims to enable complex computation verification on Bitcoin by solving two fundamental constraints: 

**State Storage Limitation**: Bitcoin's UTXO model lacks persistent state storage. 

**Computational Restriction**: Scripting language prohibits loops (non-Turing complete). 

### Core Technical Approach

![img-bitvm1](https://s3cunDa.github.io/assets/post/bitvm-1.png)

BitVM employs an OP_FraudProof mechanism (Optimistic Rollup-inspired) where on-chain verification is limited to periods of dispute and utilizes zk-SNARKs to condense computations into circuit proofs.

#### NAND Gate Implementation 

Bitcoin scripts execute base computations via **Taproot MAST (Merkelized Abstract Syntax Trees)**: 

```
\# Bitcoin Script for NAND Verification

OP_IF

 \# Validate inputs (A,B ∈ {0,1})

 OP_DUP OP_1 OP_EQUAL OP_IF OP_DROP OP_ELSE OP_0 OP_EQUALVERIFY OP_ENDIF

 OP_SWAP OP_DUP OP_1 OP_EQUAL OP_IF OP_DROP OP_ELSE OP_0 OP_EQUALVERIFY OP_ENDIF

 \# NAND Logic: OUT = ¬(A ∧ B)

 OP_BOOLAND OP_NOT 

 \# Verify claimed output

 OP_EQUALVERIFY <claimed_output> OP_EQUAL

 \# Trigger penalty if invalid

 <penalty_pubkey> OP_CHECKSIGVERIFY

OP_ENDIF
```

**All Turing-complete computations decompose into NAND gate circuits.**

### Dispute Resolution Protocol (f(x)=y Example)

#### Initialization Phase

1. **Circuit Compilation**: 

  \- Alice (Defender) and Bob (Challenger) compile `f(x)` into Boolean circuits 

  \- Generate: 

   \- Circuit Merkle root (`circuit_root`) 

   \- Pre-signed transaction set (recursive challenge paths) 

2. **Fund Locking**: 

```
  Deploy P2WSH (Pay to Witness Script Hash) contract: 

  

  Locking Script: 

   OP_HASH160 <HASH160(redeem_script)> OP_EQUAL 

  Witness: 

   <Alice_sig> <Bob_sig> <redeem_script> # 2/2 multisig 

  Output: 1 BTC locked to MAST commitment (`H_MAST = H(H(timeout_script) || H(challenge_script)`)
```

#### Challenge Protocol Workflow 


![img-bitvm2](https://s3cunDa.github.io/assets/post/bitvm2.png)

#### Key Technical Components

1. **MAST Structure**: 

  
```
  \# Branch 1: Timeout Path

  OP_CHECKSEQUENCEVERIFY <144_blocks> OP_DROP

  <Alice_pubkey> OP_CHECKSIG

  \# Branch 2: Challenge Path

  OP_IF

   <Bob_sig> OP_CHECKSIGVERIFY

   OP_SIZE <next_script_hash> OP_EQUALVERIFY

  OP_ENDIF
```

2. **Recursive Challenge Mechanism**: 

  Each UTXO output locks funds to predefined script hashes (`HASH160(script_{n+1})`): 

  *UTXO₀ → script_gate₁ → UTXO₁ → script_gate₂ → ... → UTXOₖ (atomic gate)*

### Conclusion: BitVM's Paradigm Shift 

BitVM pioneers three-layer innovation for Bitcoin: 

1. **Cryptographic Layer**: Utilizes NAND gate scripts to enable Turing-complete base functionality.
2. **Game Theory Layer**: Implements fraud proofs to convert global verification into local checks.
3. **Engineering Layer**: Leverages pre-signed transactions (TXs) and Merklized Abstract Syntax Trees (MAST) to enable state recursion.

This architecture facilitates verifiable computation without altering Bitcoin's consensus rules, thereby establishing a foundational Layer2 framework for smart contracts. Current implementations, such as Bitlayer and Citrea, showcase operational feasibility, reducing the cost of complex contract verification to practical levels.

## Examples of BitVM Applications

BitVM has the potential to unlock a wide range of new applications on Bitcoin, including:

- **Smart Contracts:** Executing more complex smart contracts than currently possible with Bitcoin's limited scripting language.
- **Rollups:** Scaling solutions for Bitcoin by enabling off-chain computation and then settling the results on-chain.
- **Gaming:** Creating provably fair and complex games directly on Bitcoin.

# BitVM Bridge

## Technical Principles

The Bitlayer blockchain employs BitVM Bridge as its core solution for enabling Bitcoin (BTC) cross-chain functionality. This bridging solution is built upon BitVM2 technology.

Unlike the original BitVM proposal, which primarily remained theoretical, BitVM2 is an engineered, implementable framework. Its goal is to translate the core concepts of BitVM into practical, runnable code and protocols.

The core design philosophy of BitVM Bridge revolves around the following key principles:

1. Off-Chain Computation: Complex and resource-intensive cross-chain verification computations are primarily shifted off the Bitcoin main chain. This is typically performed by a decentralized Verifier Network off-chain, thereby minimizing the computational load on the Bitcoin main chain and preserving its simplicity and high security.
2. Pre-Signing: Before a cross-chain operation occurs, relevant parties (typically users and verifiers) must pre-sign a series of transactions with specific conditions (such as time locks). These pre-signed transactions form the foundation for subsequent on-chain verification or dispute resolution, predefining how assets are handled in various potential scenarios. This enhances efficiency and reduces latency for on-chain interactions.
3. Computation Splitting / Circuit Representation: The complex computational logic requiring verification is broken down into smaller, verifiable logical units (often represented as binary circuits). This splitting allows the entire computation process to be divided into discrete steps, facilitating step-by-step verification within the limited capabilities of Bitcoin's scripting language.
4. On-Chain Verification (Fraud Proofs): This is the cornerstone of security. Under normal circumstances, the Bitcoin main chain does not need to verify all off-chain computations. Only when a challenger (typically an honest verifier) detects fraudulent behavior (e.g., an operator submits an incorrect result) is it necessary to submit a minimal Fraud Proof on the Bitcoin chain. This proof only needs to demonstrate an error at a specific step within the split computation to trigger penalty mechanisms defined in the pre-signed transactions, ensuring the system's honesty. This design implements Optimistic Verification, significantly boosting efficiency.

## Background

### SIGHASH

**Technical Definition****:** A Bitcoin signature hash type that precisely controls the data scope covered by a transaction signature, directly affecting the transaction's mutability and security.
**Primary Types**:

1. SIGHASH_ALL (0x01): Signs all inputs and all outputs.
2. SIGHASH_NONE (0x02): Signs all inputs but no outputs.
3. SIGHASH_SINGLE (0x03): Signs all inputs and the single output corresponding to the current input index.
4. SIGHASH_ANYONECANPAY (0x80): Combinable with above types; signs only the current input.

**Verification Mechanism****:** Validators must reconstruct an identical transaction data hash. Signature is invalid if any data mismatch occurs.

**BitVM Application****:** Enforces atomicity of multi-input operations via SIGHASH_ALL, enabling the Connector mechanism for cross-chain interoperability.

### Witness

**Technical Definition****:** A segregated transaction data structure introduced by Segregated Witness (SegWit), storing verification data (e.g., signatures, public keys) for unlocking scripts.

**Key Features**:

- Transaction Malleability Fix: Excluded from transaction ID calculation.
- Data Capacity Expansion: Supports blocks up to 4MB.
- Economic Efficiency: Witness data receives 75% fee discount (weighted in vbytes).

**Data Structure**: Each input corresponds to a Witness Stack containing ordered elements: signatures, public keys, and script parameters.

**BitVM Application**: Carries Lamport commitments, multisignature data, and SNARK proofs for lightweight on-chain verification.

### Taproot

**Technical Definition**: A privacy-enhancing spending protocol (BIP 341) combining Schnorr signatures and Merkelized Abstract Syntax Trees (MAST).

**Core Components**:

1. Internal Key: Enables key-path spending (single-signer).
2. Script Tree: Organizes conditions as Merkle-tree leaves (Tapleaf).
3. Control Block: Contains internal key + Merkle proof.

**Address Derivation**:
P = Q + hash(Q||merkle_root) × G
(Where Q = internal pubkey, G = elliptic curve base point)

**Spending Modes**:

- Key Path: Single Schnorr signature (maximizes privacy).
- Script Path: Reveals a specific Tapleaf + Merkle proof.

**BitVM Application**: Stores thousands of DisproveScripts efficiently; reveals only disputed branches during fraud challenges.

### Lamport Signatures **Technical Definition**: A post-quantum secure one-time signature scheme based on hash function one-wayness.

**Key Generation**:

- Private Key: 2×n matrix of random numbers (n = message bit length).
- Public Key: Matrix of hashes of all private key elements.

**Signing Process**:
For each message bit b_i (0 or 1):
Reveal the private key element at row i, column b_i.

**Security**:

- Single-Use: 50% private key exposed after signing.
- Quantum-Resistant: Relies solely on hash function security.

**Verification**:
Check if hashes of revealed private keys match the public key matrix.

**BitVM Application**: Enforces immutable state commitments to prevent operators from altering declared off-chain computation results.

## BitVM Bridge Design

### Overview

The simplified transaction flow of BitVM2, demonstrating how arbitrary program verification is achieved on Bitcoin. The core mechanism involves splitting a SNARK verifier program f(x) = y into k sub-programs (f1, f2, ..., fk), each small enough to execute within Bitcoin's 4MB block limit. These sub-programs form a computational chain where each intermediate state zi is computed from the previous state zi-1, starting from input z0 = x and ending with output zk = y. When a challenger disputes an operator's claim, the Challenge transaction forces the operator to reveal all intermediate states z0 through zk via the Assert transaction. If any computation step is incorrect, the challenger can prove this by executing the specific sub-program fi on-chain through a Disprove transaction, demonstrating that the claimed zi does not equal fi(zi-1). This elegant design enables complex computations to be verified on Bitcoin using only three on-chain transactions, with the full SNARK verification happening off-chain and only the disputed step being executed on-chain when necessary.

![img](https://s3cunDa.github.io/assets/post/bitvmbridge.png)

#### User Perspective

**Scenario**

Alice wants to cross 1 BTC to another chain(BTC -> wBTC in ETH). Alice transfers 1 wBTC to Bob in ETH. Then Bob wants to cross 1 BTC from another chain(wBTC -> BTC).

**Step1: Alice create PegIn tx**

```

tx_pegin_001

Input[0]:

\- prevout: Alice UTXO (txid:index)

\- scriptSig: None（SegWit）

\- sequence: 0xfffffffe

\- witness:

 \- Alice sig (SIGHASH_ALL)

 \- Alice PubK

Output[0]:

\- value: 1 BTC (100000000 satoshis)

\- scriptPubKey: OP_n <pubkey1> <pubkey2> ... <pubkeyn> OP_n OP_CHECKMULTISIG (P2WSH include n-of-n multisig hash script)

```

fee: 0.001 BTC (pay by Alice)

**Step2: Side chain mint 1 wBTC to Alice account.**

**Step3: Alice transfers 1 wBTC to Bob account.**

**Step4: Bob burns 1 wBTC and initiates a cross chain event on the side chain.**

**Step5: Operator initiates PegOut tx**

```
tx_pegout_001

 Input[0]:

  \- prevout: any Bob UTXO (as a trgger)

  \- scriptSig: None

  \- sequence: 0xfffffffe

  \- witness:

   \- Bob sig (SIGHASH_SINGLE|ANYONECANPAY)

   \- Bob Pubk

 Input[1]:

  \- prevout: Operator 1BTC UTXO

  \- scriptSig: None

  \- sequence: 0xfffffffe

  \- witness:

   \- Operator sig (SIGHASH_ALL)

   \- Operator PubK

 Output[0]:

  \- value: 0.98 BTC (0.02 BTC fee)

  \- scriptPubKey: OP_DUP OP_HASH160 <Bob Pubk Hash> OP_EQUALVERIFY OP_CHECKSIG

fee: 0.001 BTC

```



#### Operator perspective

##### *Happy Path*

The operator needs to reclaim 1 BTC and reward.

**Step1: Initiate Kickoff tx - Start Light Client Verification**

```
tx_kickoff_001

 Input[0]:

  \- prevout: Any Operator UTXO (1 BTC)

  \- scriptSig: None

  \- sequence: 0xfffffffe

  \- witness:

   \- Operator sig (SIGHASH_ALL)

   \- Operator pubk

 Output[0]: 

  \- value: 1 BTC

  \- scriptPubKey: Taproot

  Taproot Tree：

   Leaf1: <2000> OP_CSV OP_DROP # wait 2000 block（ob window）

​       <n Covenant pubk> OP_CHECKSIGVERIFY # must verified by federation

​       <pk_SBO> OP_CHECKLAMPORT # super block commitment

   

   Leaf2: <20> OP_CSV OP_DROP # 20 block time out（~3h）

​       <Covenant> # start time

   

   Leaf3: <2020> OP_CSV OP_DROP # 2020 block time out

​       <Covenant check> # claim time

 Output[1]: # connector

  \- value: 0 satoshis

  \- scriptPubKey: Taproot

  Taproot tree：

   Leaf1: Same as Output[0] Leaf2（start time）

   Leaf2: <Operator pubk> OP_CHECKSIGVERIFY # Only Operator

​       <pk_TS> OP_CHECKLAMPORT # Must commit timestamp

​       <750000> OP_CLTV # lock for a long time to prevent selfish mining

   fee: 0.001 BTC
```

**Step2 Start time**

```
tx_starttime_001

Input[0]:

\- prevout: tx_kickoff_001:1

\- scriptSig: None

\- sequence: 0xfffffffe

\- witness:

 \- Operator sig（SIGHASH_ALL）

 \- timestamp）

 \- Lamport commitment c_TS (for time stamp)

 \- Operator pubk

 \- control block（Leaf2）

Output[0]:

\- value: 0 satoshis

\- scriptPubKey: OP_RETURN (burn)

**Step3 ClaimLC**

tx_claimlc_001

 Input:

  \- prevout: tx_kickoff_001:0

  \- scriptSig: None

  \- sequence: 2000（must wait for 2000 block）

  \- witness:

   \- Covenant n-of-n 

   \- Lamport commitment c_SBO[32 element] # lock super block hash

   \- Super block SBO：

​    {

​     hash: "00000000000000000000000123abc...", 

​     height: 751234,

​     time: 1234567890,

​     difficulty: 0x170365a1

​    } # prove found heaviest block in 2000 blocks

   \- control block（Merkle proof） # use leaf1 path

 Output[0]: #dispute control

  \- value: 1 BTC

  \- scriptPubKey: new Taproot

  tap tree：

   Leaf1: (happy path)

​    <100> OP_CSV OP_DROP # 100 block challendge window

​    <Covenant> 

   Leaf2: (side chain challendge path)

​    <20> OP_CSV OP_DROP # wait 20 block

​    AssertScript:

​     <pk_z0> OP_CHECKLAMPORT # SNARK input

​     <pk_z1> OP_CHECKLAMPORT # step 1

​     ...

​     <pk_z1000> OP_CHECKLAMPORT # final result

​     \# must reveal every path once challenged

   Leaf3: (super block challendge path)

​    <pk_SBO> OP_CHECKLAMPORT # superblock data

​    <pk_TS> OP_CHECKLAMPORT # time stamp

​    verification script：

​     if SB_V.weight > SBO.weight # challenger has heavier block

​     and TS < SB_V.time < TS+2000 # within time window

​     then slash 

 Output[1]: (connector)

  \- value: 0 satoshis

  \- scriptPubKey: P2PKH(Operator pubk)
```


**Step4 PayoutOptimistic**

```

 Input[0]: 

  \- prevout: tx_claimlc_001:0

  \- scriptSig: None

  \- sequence: 100(challenge window)

  \- witness:

   \- Covenant

   \- Operator sig SIGHASH_ALL

​    \- to force sign every part of tx

​    \- must include input

​    \- if connector was spend then invalid

   \- control block(leaf 1）

 Input[1]: (connector)

  \- prevout: tx_claimlc_001:1

  \- scriptSig: None

  \- sequence: 0xfffffffe

  \- witness:

   \- Operator sig（SIGHASH_ALL）

​    must same as input 0

   \- Operator pubk

Input[2]: (PegIn output - Alice locked asset) 

\- prevout: tx_pegin_001:0 

\- scriptSig: None

\- sequence: 0xfffffffe 

\- witness: 

\- Covenant 

\- Operator sig（SIGHASH_ALL） 

\- other

 Output[0]:

  \- value: 1.995 BTC

  \- scriptPubKey: P2PKH

fee: 0.005 BTC

```

##### Unhappy Path (BTC)

###### *Operator didnt initiate StartTime on time*

```
tx_timetimeout_001

Input[0]:

 \- prevout: tx_kickoff_001:0

 \- scriptSig: None

 \- sequence: 20（timelock）

 \- witness:

  \- Covenant

  \- control block（Leaf2）

 script（KickOff output Leaf2）：

 <20> OP_CSV OP_DROP <Covenant>

Input[1]: (connector)

 \- prevout: tx_kickoff_001:1

 \- scriptSig: None

 \- sequence: 20（timelock）

 \- witness:

  \- Covenant

  \- control block（Leaf1）

Output[0]: (burn)

 \- value: 0.7 BTC

 \- scriptPubKey: OP_FALSE

Output[1]: 

  \- value: 0.3 BTC

  \- scriptPubKey: <c pubk>
```

###### *Operator didnt initiate ClaimLC on time*

```
tx_claimtimeout_001

 Input[0]:

  \- prevout: tx_kickoff_001:0

  \- scriptSig: None

  \- sequence: 2020（2000+20）

  \- witness:

   \- Covenant

   \- Control block（Leaf3）

  script（KickOff output Leaf3）：

  <2020> OP_CSV OP_DROP

  <Covenant>

 Output[0]: (burn)

  \- value: 0.7 BTC

  \- scriptPubKey: OP_FALSE

 Output[1]: 

  \- value: 0.3 BTC

  \- scriptPubKey: <c pubk>
```

###### *Challenge super block*

```
tx_disapprovechain_001

 Input[0]:

  \- prevout: tx_claimlc_001:0

  \- scriptSig: None

  \- sequence: 0xfffffffe

  \- witness:

   \- Covenant

   \- challenger super block SB_V：

​    {

​     hash: "00000000000000000000000000789...", # more prefix zero

​     height: 751567,

​     time: 1234568900,

​     difficulty: 0x170365a1

​    }

   \- Operator SB_O（copy from ClaimLC）

   \- Lamport commitment c_SBO（copy from ClaimLC）

   \- TS = 750000

   \- Lamport commitment c_TS

   \- control block（Leaf3）

  script（ClaimLC output Leaf3）：

  <pk_SBO> OP_CHECKLAMPORT

  <pk_TS> OP_CHECKLAMPORT

   

  if：

1. SB_V.hash < SB_O.hash（heavier)
2. SB_V.time > TS（within time window）
3. SB_V.time < TS + 2000

  pass

 Output[0]: 

  \- value: 0.7 BTC

  \- scriptPubKey: OP_FALSE

   

 Output[1]: 

  \- value: 0.3 BTC

  \- scriptPubKey: <c pubk>
```

##### Unhappy Path (side chain)

**Step1: Challenger initiate Challenge tx**

```
tx_challenge_001

 Input[0]: (connector)

  \- prevout: tx_claimlc_001:1

  \- scriptSig: None

  \- sequence: 0xfffffffe

  \- witness:

   \- Operator sig

   \- Operator pubk

  script：

  OP_DUP OP_HASH160 <Operator pubk hash> OP_EQUALVERIFY OP_CHECKSIG

 Input[1]: (challenger BTC)

  \- prevout: challenger UTXO

  \- scriptSig: None 

  \- sequence: 0xfffffffe

  \- witness:

   \- challenger sig

   \- challenger pubk

 Output[0]:

  \- value: 0.1 BTC (challenge fee)

  \- scriptPubKey: P2PKH(Operator)
```

**Step2: Operator initiate Assert tx - reveal entire compute path to self prove**

```
tx_assert_001

 Input[0]:

  \- prevout: tx_claimlc_001:0

  \- scriptSig: None

  \- sequence: 20

  \- witness:

   \- Covenant

   \- z'0：part of SNARK input（no spb）

   \- z1 - z1000：all intermidiate state

   \- c_z'0：z'0 Lamport commitment

   \- c_z1 - c_z1000：all state Lamport commitment

   \- SBO：super block（copy from ClaimLC）

   \- c_SBO：super block Lamport Commitment

   \- TS

   \- c_TS：TS Lamport Commitment

   \- Control block（Leaf2）

  unlock script（ClaimLC AssertScript）：

  <20> OP_CSV OP_DROP

  <Covenant>

  <pk_z'0> OP_CHECKLAMPORT

  <pk_z1> OP_CHECKLAMPORT

  ...

  <pk_z1000> OP_CHECKLAMPORT

 Output[0]:

  \- value: 1 BTC + 0.1 BTC = 1.1 BTC

  \- scriptPubKey: Taproot

  Taproot tree（1001 leaf）：

   DisproveScript1： z0 → z1

   DisproveScript2： z1 → z2

   ...

   DisproveScript1000： z999 → z1000

   PayoutScript：<100> OP_CSV <Covenant> <Operator sig>

fee: 0.01 BTC（big tx）

~400KB（reach limit）
```

**Step3: challenger unlock** tx_assert_001:0**, challenger wins**

```
tx_disprove_001

 Input[0]:

  \- prevout: tx_assert_001:0

  \- scriptSig: None

  \- sequence: 0xfffffffe

  \- witness:

   \- Covenant 

   \- zi-1: Input of the erroneous step (e.g., z41)

   \- zi: Output claimed by the operator (e.g., z42)

   \- c_zi-1: Lamport commitment of the input

   \- c_zi: Lamport commitment of the output

   \- If i=1: SBO, TS, and their commitments are also required

   \- Control block (proof uses DisproveScript_i)

  Script executed (DisproveScript_i):

  	# Verify commitment

  	CheckLampCommVerify(pk_zi-1, zi-1, c_zi-1)

  	CheckLampCommVerify(pk_zi, zi, c_zi)

  	# Execute calculation

  	zi_correct = fi(zi-1)

  	# Compare results

  	if (zi != zi_correct) return TRUE

 Output[0]: (Burn)

  \- value: 0.7 BTC

  \- scriptPubKey: OP_FALSE

 Output[1]: (Challenger Reward)

  \- value: 0.4 BTC (0.3 + 0.1 deposit refund)

  \- scriptPubKey: <Challenger Address>

Transaction Fee: Approximately 0.002 BTC

SIGHASH: SIGHASH_SINGLE
```

**Step3: challenger cannot unlock** tx_assert_001:0**, eg exceed time window, operator wins**

```
tx_payout_001

 Input[0]:

  \- prevout: tx_assert_001:0(~1.1btc)

  \- scriptSig: empty

  \- sequence: 100 (relative timelock)

  \- witness:

   \- Covenant 

   \- Operator's signature

   \- control block (using PayoutScript)

  Unlock Script (last Tapleaf):

  <100> OP_CSV OP_DROP

  <Covenant check>

  <O1 public key> OP_CHECKSIG

 Input[1]: (PegIn funds)

  \- prevout: tx_pegin_001:0 (1btc)

  \- scriptSig: empty

  \- sequence: 0xfffffffe

  \- witness:

   \- Covenant signatures [n]

   \- O1's signature

Outputs:

 Output:

  \- value: 2.08 BTC (after deducting transaction fees)

  \- scriptPubKey: P2PKH(O1 address)

Transaction Fee: approx. 0.005 BTC
```


# Conclusion

BitVM Bridge ingeniously leverages Bitcoin's native properties to implement a trust-minimized cross-chain bridge that is feasible for real-world deployment. Its distinguishing feature compared to other Layer 2 solutions is the enablement of general computation verification on Bitcoin, allowing complex cross-chain validation logic to execute directly on Layer 1, rather than merely using Bitcoin as a 32-byte data availability layer. While this design demonstrates significant technical innovation potential, the current implementation faces several challenges:

1. **Committee Centralization Risk**: All computational verification processes depend on the Covenant signing committee, which possesses excessive authority and constitutes a centralization bottleneck in the system.
2. **Computational Resource Centralization**: SNARK circuit generation and verification script construction require extremely high computational power, creating another layer of centralization in practice.
3. **High Resource Threshold**: The computational, storage, and on-chain transaction costs required for system operation are prohibitively high, making it primarily suitable for large-value cross-chain transactions.
4. **Limited Fund Composition Flexibility**: During the PegIn and PayoutOptimistic phases, the pre-signing mechanism restricts flexible fund combinations, preventing adaptation to dynamic payment requirements.
5. **Liquidity Management Challenges**: When PegIn and PegOut demands are imbalanced, operators require substantial liquid capital for fronting payments, potentially causing system liquidity pressure and capital inefficiency.

# References

- [Technical Whitepaper for BitVM] https://bitvm.org/bitvm_bridge.pdf 
- [Technical Whitepaper for Bitlayer] https://docs.bitlayer.org/docs/BitVMBridge/protocol 
- [BitVM Bridge] https://bitvm.org/bitvm_bridge.pdf
- [BTC] https://learnmeabitcoin.com/technical/ 