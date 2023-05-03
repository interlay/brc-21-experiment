---
description: Proposed standard for cross-chain BRC-20 tokens
---

# BRC-21 Experiment

### Introduction

This proposal outlines a standard to mint and redeem BRC tokens to/from Bitcoin that were first created on other "source" chains, such as Ethereum, Cosmos, Polkadot, or Interlay.&#x20;

**Technical Requirements**

* Custom indexer that validates the BRC-21 mint, transfer, and redeem history
* BTC-Relay: a BTC-light client implemented as a smart contract on the "source" chain, capable of verifying of inclusion of BTC transactions and parsing them. See e.g. [here](https://spec.interlay.io/spec/btc-relay/index.html) for a specification. &#x20;



### Protocol

**Deploy**

The deploy operation stays similar to that of BRC-20 tokens, with minor modifications.&#x20;

* "max" field is made optional: max supply is defined on the source chain. Specifying it in the BRC-21 deployment is hence optional but can serve as a failsafe.&#x20;
* "lim" field introduced in the BRC-20 standard is removed for scalability reasons. Since BRC-21 tokens follow strict mint and redeem rules, there is no need to impose limits on how many tokens can be minted in a single transaction.
* "src" field added to specify the chain from which we are "importing" the tokens. Can be a string such as "Ethereum" or a unique integer identifier (would require an agreed-upon directory)
* "cont" field added to act as unique identifier of the token on the source chain, e.g. the ERC-20 contract address on Ethereum

```
{ 
"p": "brc-21", 
"op": "deploy", 
"tick": "COIN", 
"max": "10000000" (optional), 
“src”: “CHAIN”,
“cont”: “contract-address-on-CHAIN” 
}
```



**Mint**

1. Create BRC-20 mint transaction on Bitcoin (inscription). e.g. 100 BRC-20 COIN
2. Lock 100 COIN on CHAIN and pass the inscription ID to the contract
3. Indexers will now accept the BRC-20 COIN as _valid_

```
{
"p": "brc-21",
"op": "mint",
"tick": "COIN",
"amt": "100",
"src": "CHAIN"
}
```

#### **Transfer**

Transfer operation is same as in the BRC-20 protocol.

```
{ 
  "p": "brc-21",
  "op": "transfer",
  "tick": "COIN",
  "amt": "100",
}
```

**Redeem**

Introduces a new “Redeem” BRC-20 operation that triggers redemption of the equivalent amount of underlying COIN tokens on the target chain.

```
{ 
"p": "brc-21", 
"op": "redeem", 
"tick": "COIN", 
"amt": "1000",
“dest”: “CHAIN”,
“acc”: “account-on-CHAIN” 
}
```

1. Inscribe “Redeem” specifying _dest_, _acc_, and _amount_.
2. Submit BTC tx inclusion proof to [BTC-Relay](https://spec.interlay.io/spec/btc-relay/index.html) deployed on CHAIN&#x20;
3. Contract on CHAIN allocates the COINs to the account specified by _acc_
