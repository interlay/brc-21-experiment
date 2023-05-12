---
description: >-
  Proposal for a fully decentralized protocol to bridge BRC-20 and similar
  tokens to other chains
---

# BRC-42: A Cross-Chain Standard For BRC-20 Tokens

_Version 0.1 - May 11th 2023_

### Introduction

This proposal outlines **an extension to the BRC-20 standard** that allows bridging BRC-20 tokens to chains with smart contract support, such as Ethereum, Interlay, Solana, Polkadot, etc.&#x20;

The proposed method does not require trust in any 3rd party.&#x20;

While the proposed protocol is specified for BRC-20 tokens, it can be extended to support any other token standard that is built on top of Bitcoin, including inscriptions, colored coins, and Taro.&#x20;

#### **Technical Requirements**

* **Custom indexer** that validates the BRC-20 cross-chain "bridge-out" and "bridge-in" operations on Bitcoin, as well as the contract state on the destination chain. Indexers track the total amount of "bridged out" tokens for each tracked BRC-20 token.&#x20;
* **Destination chain contract** that handles bridge-out and bridge-in operations on the destination chain.&#x20;
* **BTC-Relay**: a BTC-light client implemented as a smart contract on the destination chain, capable of verifying the mainchain inclusion of BTC transactions and parsing them. See e.g. [here](https://spec.interlay.io/spec/btc-relay/index.html) for a specification,  [here](https://github.com/interlay/btc-relay-solidity) for a PoC implementation on Ethereum, and [here](https://github.com/interlay/interbtc/tree/master/crates/btc-relay) for an audited production implementation in Rust on Interlay. &#x20;

#### Use Cases

The main use case for BRC-20 tokens is to unlock access to decentralized financial protocols. Trading on decentralized AMMs, using them as collateral in lending protocols, creating auctions, etc. &#x20;

Building decentralized financial primitives is difficult on Bitcoin and often limited to very basic products like atomic swaps. Allowing BRC-20s to be used on other chains that have smart contract support and advanced DeFi ecosystems can be beneficial for the growth of BRC-20s and future experiments.&#x20;

### Definitions

* `CHAIN`: the destination chain, e.g. Ethereum or Interlay
* `COIN`: BRC-20 token, to be minted on the destination chain
* `wCOIN`: "wrapped" representation of the BRC-20 token on the destination chain,  e.g. an ERC-20 token on Ethereum
* `CONTRACT`: smart contact on CHAIN that handles mint and redeem operations of `wCOIN` on `CHAIN`
* `BTC-RELAY`: smart contract on CHAIN that is capable of verifying the Bitcoin main chain (incl. difficulty adjustments), verifying transaction inclusion, and parsing transaction data (specifically, extracting the BRC-20 witness data from P2TR transactions).

### Protocol

This proposal introduces two new operations to the BRC-20 standard:&#x20;

* **Bridge-out**: burns `COIN`BRC-20 tokens on Bitcoin and mints the equivalent amount on the destination chain `CHAIN`.&#x20;
* **Bridge-in**: burns `wCOIN` wrapped BRC-20 tokens on the destination chain `CHAIN`and mints the equivalent amount of `COIN` tokens on Bitcoin

_No changes are needed to the deploy, mint, and transfer operations._&#x20;

#### **Deploy on** destination **chain**

The deployment procedure depends on the implementation of the source chain. Requirements of the `CONTRACT` smart contract are:&#x20;

* Mint `wCOIN` tokens if and only if a valid transaction inclusion proof was received for a "bridge-out" operation of X `COIN` tokens on Bitcoin. Use `BTC-RELAY` to verify and validate (i.e. parse) the corresponding Bitcoin transaction.&#x20;
* If an account that owns Y `wCOIN` tokens requests to bridge back X `wCOIN`to Bitcoin, such that $$X \leq Y$$: Burn (remove from circulation) X `wCOIN` tokens and emit a "bridge-in" event.&#x20;

This assumes a `BTC-RELAY` contract is already deployed on `CHAIN` or that `CHAIN` has built-in Bitcoin light client functionality. For requirements, see [here](https://spec.interlay.io/).&#x20;

#### **Bridge-out**

Locks ("bridges-out") X `COIN` BRC-20 tokens on Bitcoin and mints X `wCOIN` on `CHAIN`.

The "bridge-out" operation is defined as follows:

```
{ 
"p": "brc-21", 
"op": "bridge-out", 
"tick": "COIN", 
"amt": "1000",
“dest”: “CHAIN”,
“acc”: “account-on-CHAIN” 
}
```

| Key                                   | Required? | Description                                                                                                                                                                |
| ------------------------------------- | --------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| p                                     | Yes       | Protocol: Helps other systems identify and process brc-20 events                                                                                                           |
| op                                    | Yes       | Operation: Type of event (Deploy, Mint, Transfer, Bridge-out, Bridge-in)                                                                                                   |
| tick                                  | Yes       | Ticker: 4 letter identifier of the brc-20                                                                                                                                  |
| amt                                   | Yes       | Amount to transfer: States the amount of the brc-20 to bridge-out.                                                                                                         |
| dest                                  | Yes       | Identifier specifying the destination chain `CHAIN`. Format: tbd. Can be verbose (e.g. "Interlay") or an integer specified in a global registry agreed upon by indexers.   |
| <mark style="color:green;">acc</mark> | Yes       | Account on the destination chain `CHAIN`, in the respective format. Max. length: tbd (32 bytes?)                                                                           |



1. Inscribe “bridge-out” operation on Bitcoin, specifying `amt`, `dest` and `acc`.
2. Submit BTC transaction and transaction inclusion proof ("SPV proof") to `CONTRACT` which in turn calls `BTC-RELAY` to complete the verification.
3. If `BTC-Relay` returns "true" (i.e., the transaction is verifiably included in the Bitcoin main chain), the `CONTRACT` mints X `wCOIN` to the account specified in the `acc` field, and emits a "Mint" event.&#x20;

_Indexer behavior_

* Indexers MUST verify that the "Mint" event emitted by the `CONTRACT` on `CHAIN` matches the `amt`, `dest`, and `acc` fields of the bridge-out operation
* Indexers MUST mark the X `COIN` tokens as "bridged-out" and increase the total amount of  `COIN` BRC-20 tokens marked as "bridged-out" by X
* Indexers MUST NOT allow transfer (or any other) operations of BRC-20 tokens marked as "bridged-out"
* _Consideration for future versions of BRC-20 tokens: Indexers MUST NOT remove the X `COIN` tokens from the total supply of `COIN`_

#### **Bridge-in**

Burns X `wCOIN` tokens on `CHAIN` and mints ("bridges-in") X `COIN` BRC-20 tokens on Bitcoin - if and only if X is less or equal to the total amount of `COIN` tokens tracked as "bridged-out".&#x20;

The "bridge-out" operation is defined as follows:

```
{ 
"p": "brc-20", 
"op": "bridge-in", 
"tick": "COIN", 
"amt": "1000",
“src”: “CHAIN”
}
```

| Key  | Required? | Description                                                                                                                                                                  |
| ---- | --------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| p    | Yes       | Protocol: Helps other systems identify and process brc-20 events                                                                                                             |
| op   | Yes       | Operation: Type of event (Deploy, Mint, Transfer, Bridge-out, Bridge-in)                                                                                                     |
| tick | Yes       | Ticker: 4 letter identifier of the brc-20                                                                                                                                    |
| amt  | Yes       | Amount to transfer: States the amount of the brc-20 to bridge-in.                                                                                                            |
| src  | Yes       | Identifier specifying the "source" chain `CHAIN` = the "destination" chain of a previous bridge-out operation. Format: same as the `dest` field in the bridge-out operation. |



1. Inscribe “bridge-in” operation on Bitcoin, specifying `amt` and `dest`.
2. Lock X `wCOIN` tokens on in the `CONTRACT` on `CHAIN` and pass the inscription or transaction ID of the transaction containing the "bridge-in" operation performed in step (1). The `CONTRACT` verifies that X =`amt`. If correct, emits a "bridge-in" event and burns the `wCOIN` tokens at the end of the operation, removing them from circulation.

_Indexer behavior_

* Indexers MUST verify that the event emitted by the `CONTRACT` on `CHAIN` matches the `amt` and `src` fields of the bridge-in operations
* Indexers MUST set the `COIN` balance of the address associated with the bridge-in txid to X
* Indexers MUST reduce the tracked total amount of "bridged-out" `COIN` tokens by X

## Changelog

* May 11th 2023 23:50 UTC: version 0.1 completed
