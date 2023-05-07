---
description: >-
  Proposal for minting & redeeming fully decentralized, cross-chain BRC-20
  tokens
---

# BRC-21 Token Standard

_Version 0.1 - May 7th 2023_

### Introduction

This proposal outlines a standard to mint and redeem BRC-20 tokens to/from Bitcoin that were first created on other "source" chains, such as Ethereum (ETH, DAI, ...), Cosmos, Polkadot, or Interlay.

The proposed method does not require trust in any 3rd party.&#x20;

#### **Technical Requirements**

* **Custom indexer** that validates the BRC-21 mint, transfer, and redeem operations om Bitcoin, as well as the contract state on the SOURCE chain.&#x20;
* **SOURCE chain contract** that handles mint and redeem operations on the SOURCE chain.&#x20;
* **BTC-Relay**: a BTC-light client implemented as a smart contract on the "source" chain, capable of verifying of inclusion of BTC transactions and parsing them. See e.g. [here](https://spec.interlay.io/spec/btc-relay/index.html) for a specification,  [here](https://github.com/interlay/btc-relay-solidity) for a PoC implementation on Ethereum, and [here](https://github.com/interlay/interbtc/tree/master/crates/btc-relay) for an audited production implementation in Rust on Interlay. &#x20;

#### Use Cases

While using BRC-21 assets to represent ETH, DOT, SOL etc. on Bitcoin is possible, we believe the main use case that will emerge from this standard is **deploying decentralized stablecoins onto Lightning Network or similar payment protocols**.&#x20;

Decentralized stablecoins (read [this paper](https://arxiv.org/abs/2006.12388) for a good overview of different models) like MakerDAO, Liquity, or RAI require sophisticated mint, redeem, and liquidation mechanisms to retain their peg. These protocols cannot be deployed directly on Bitcoin due to lack of programmability - and it is unlikely that this will change in the near future. Instead of investing time in engineering protocols onto Bitcoin that can be represented by a few lines of code elsewhere, we believe this fully decentralized method of minting and redeeming cross-chain assets to/from Bitcoin has a much higher chance of achieving mass adoption.&#x20;

### Definitions

* `CHAIN`: the source chain, e.g. Ethereum or Interlay
* `COIN`: token created/hosted on the source chain, to be minted as BTC-21 token on Bitcoin
* `bCOIN`: BRC-21 representation of COIN
* `CONTRACT`: smart contact on CHAIN that handles mint and redeem operations of COIN on CHAIN
* `BTC-RELAY`: smart contract on CHAIN that is capable of verifying the Bitcoin main chain (incl. difficulty adjustments), verifying transaction inclusion, and parsing transaction data (specifically, extracting the BRC-21 witness data from P2TR transactions).

### Protocol

#### **Deploy on Bitcoin**

The `deploy` operation stays similar to that of [BRC-20 tokens](https://domo-2.gitbook.io/brc-20-experiment/), with minor modifications.&#x20;

* `max` field is made optional: max supply is defined on the source chain. Specifying it in the BRC-21 deployment is hence optional but can serve as a failsafe.&#x20;
* `lim` field introduced in the BRC-20 standard is removed for scalability reasons. Since BRC-21 tokens follow strict mint and redeem rules, there is no need to impose limits on how many tokens can be minted in a single transaction.
* `src` field added to specify the chain from which we are "importing" the tokens. Can be a string such as "Ethereum" or a unique integer identifier (would require an agreed-upon directory)
* `id` field added to act as unique identifier of the token on the source chain, e.g. the ERC-20 contract address on Ethereum. Alternatively, this can be a unique ID assigned to the ERC-20 token by `CONTRACT`

```
{ 
"p": "brc-21", 
"op": "deploy", 
"tick": "bCOIN", 
"max": "10000000" (optional), 
“src”: “CHAIN”,
“id”: “contract-address-on-CHAIN” 
}
```

#### **Deploy on SOURCE chain**

The deployment procedure depends on the implementation of the source chain. Requirements of the `CONTRACT` smart contract are:&#x20;

* Lock `COIN` token and emit "Lock" event
* Unlock `COIN` token if and only iff a corresponding "redeem" operation was included in a transaction on Bitcoin. Uses `BTC-RELAY` to verify and validate (i.e. parse) the corresponding Bitcoin transaction.&#x20;

This assumes a `BTC-RELAY` contract is already deployed on `CHAIN` or that `CHAIN` has built-in Bitcoin light client functionality. For requirements, see [here](https://spec.interlay.io/).&#x20;

#### **Mint**

Locks X `COIN` tokens on `CHAIN` and mints X `bCOIN` BRC-21 tokens on Bitcoin

The "mint" operation on Bitcoin is defined as follows (inscription):&#x20;

```
{
"p": "brc-21",
"op": "mint",
"tick": "bCOIN",
"amt": "100",
"src": "CHAIN"
}
```

1. Inscribe a BRC-21 mint operation on Bitcoin, specifying the amount to be mined in the `amt` field and the source chain in the `src` field, e.g. 100 `bCOIN` from Ethereum
2. Lock 100 `COIN` on `CHAIN` and pass the inscription or transaction ID to the `CONTRACT` alongside a transaction inclusion proof. _The simplest way to achieve this is to submit the raw Bitcoin transaction and the Merkle-Tree path proving inclusion in a Bitcoin block to `CONTRACT` which in turn calls `BTC-RELAY` to complete the verification._&#x20;
3. Indexers will now accept the BRC-21 `bCOIN` as _valid_

_Note: we create the Bitcoin transaction first as this is the easiest way to create a unique, 1:1 mapping between Bitcoin and the CONTRACT on CHAIN. Alternatively, a "request" transaction can be made on CHAIN, generating a unique ID that must be included alongside the Mint transaction on Bitcoin._&#x20;

#### Transfer

Transfer operation remains the same as specified in the BRC-20 standard.

```
{ 
  "p": "brc-21",
  "op": "transfer",
  "tick": "bCOIN",
  "amt": "100",
}
```

#### Redeem

Burns X `bCOIN` tokens on Bitcoin and unlocks X `COIN` on `CHAIN`.

The "redeem" operation is defined as follows:

```
{ 
"p": "brc-21", 
"op": "redeem", 
"tick": "bCOIN", 
"amt": "1000",
“dest”: “CHAIN”,
“acc”: “account-on-CHAIN” 
}
```

The `dest` field specifies the destination `CHAIN` and MUST be the same as the `src` field in the mint operation. The `acc` field specifies the recipient account of `COIN` on `CHAIN`.

1. Inscribe “redeem” operation on Bitcoin, specifying `amt`, `dest` and `acc`
2. Submit raw BTC transaction and transaction inclusion proof to `CONTRACT` which in turn calls `BTC-RELAY` to complete verification
3. If `BTC-Relay` returns "true", the `CONTRACT` unlocks X `COIN` to the account specified in the `acc` field. BRC-21 indexers will now consider the X `bCOIN` as burned and will no longer track them on Bitcoin.&#x20;

## Changelog

* May 7th 2023 12:40 UTC: version 0.1 completed
