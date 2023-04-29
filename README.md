---
description: Propsed standard for cross-chain BRC-20 tokens
---

# BRC-21 Experiment

### Introduction

This proposal outlines a standard to mint and redeem BRC-20 tokens (now BRC-21) that were first created on other "source" chains, such as Ethereum, Cosmos, Polkadot, or Interlay.

**Requirements**

* Custom indexer that validates the BRC-21 mint, transfer, and redeem history
* BTC-Relay: a BTC-light client implemented as a smart contract on the "source" chain, capable of verifying of inclusion of BTC transactions and parsing them. See e.g. [here](https://spec.interlay.io/spec/btc-relay/index.html) for a specification. &#x20;



### Protocol



**Deploy**

{&#x20;

"p": "brc-20",&#x20;

"op": "deploy",&#x20;

"tick": "COIN",&#x20;

"max": "21000000" (optional),&#x20;

~~"lim": "1000",~~

“src”: “CHAIN”,

“cont”: “contract-address-on-CHAIN”&#x20;

}



**Mint**

1. Create BRC-20 mint transaction on Bitcoin (inscription). e.g. 100 BRC-20 COIN
2. Lock 100 COIN on CHAIN and pass the inscription ID to the contract
3. Indexers will now accept the BRC-20 COIN as _valid_

**Redeem**

Uses a new “Redeem” BRC-20 operation that triggers redemption of the equiv. amount of underlying COIN tokens on the target chain.

{&#x20;

"p": "brc-20",&#x20;

"op": "redeem",&#x20;

"tick": "COIN",&#x20;

"amt": "1000",

“dest”: “CHAIN”,

“acc”: “account-on-CHAIN”&#x20;

}

1. Inscribe “Redeem” specifying _dest_, _acc_, and _amount_.
2. Submit BTC tx inclusion proof to [BTC-Relay](https://spec.interlay.io/spec/btc-relay/index.html) deployed on CHAIN&#x20;
3. Contract on CHAIN allocates the COINs to the account specified by _acc_
