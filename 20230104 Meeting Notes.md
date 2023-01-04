# Public Mempool Kickoff
January 4, 2023

# Context

Bundler implementations are in early stages and only support private mempools. A decentralized mempool is critical to the health and adoption of EIP-4337. We are meeting to agree on the protocol to implement a shared mempool.

# State of Current Bundlers

- Infinitism’s reference implementation is in typescript
- Nethermind has two ERC-4337 implementations. ~6 months out of date but implements a p2p protocol to broadcast UserOperations similar to devp2p
- Stackup’s Golang implementation is up to date but only supports private mempools
- Vid’s Rust implementation is in very early stages, Rust libraries not up to date.
- Etherspot is doing a typescript based bundler, very early stage
- Biconomy’s implementation is node.js
- Candide’s Python implementation is early but running

# Design Objectives

- ERC-4337 mempool should be as close to Ethereum mainnet mempool as possible. We don’t need to reinvent the wheel.
    - Only deciding on P2P networking protocol
    - Bundler architecture is out of scope and up to implementations.
- Yoav sent a [report from Quilt team](https://ethresear.ch/t/dos-vectors-in-account-abstraction-aa-or-validation-generalization-a-case-study-in-geth/7937), who thought about a lot of these problems 2 years ago.
- In general, EIP-4337 is designed to force an attacker to do O(n) work. Only cases where this is violated is when actors need to stake.
- We need to add new messages type that doesn’t exist in Ethereum mempool. Ethereum only cares if nonce and signature are valid - however 4337 requires a simulation to know if a node is spamming. So far, we know we need:
    - **Block hash:** Used by bundlers to simulate a UserOperation against the state at the given block. If it fails then the sender must be a spammer and should be booted.
    - **Mempool ID.** EIP-4337 supports the use of an alternate mempool that bundlers are able to opt into which is seperate from the canonical mempool.
- Interoperability with non-Ethereum EVMs is highly desired. We want to have the same RPC - or as close to it - for other EVMs. For this reason the bundler p2p mempool should not be tightly coupled with the network mempool. A bundler mempool that is tightly coupled to the Ethereum mempool may get into problems with networks that have sequencers, reorgs, and so on.
- All bundler implementations should pass all tests in the Infinitism test suite, but don’t necessarily need to in order to participate in the bundler mempool. Need an integration test for p2p protocol.

# Other considerations

Other things that were discussed that are related to Bundlers but not necessarily the networking protocol

- **********Prioritization factors.********** i.e. How Bundlers are expected to order UserOperations in a bundle.
    - This is up to the implementation to find the best optimal block.
    - Should support the naive case of ordering by priority fees as that’s what the test suite uses.
- **Relationship between Bundlers and Ethereum nodes**
    - Bundlers should be part of the node. This does not mean it needs to be in the same executable but at least co-located.
    - Bundlers need low latency access to a node in order to carry out simulations (trace calls) against the current state.

# Actions

- Partha, Yoav, Hazim, and Jorge (if he agrees) start working on message type
- John send out meeting invitation for 3 weeks from today
