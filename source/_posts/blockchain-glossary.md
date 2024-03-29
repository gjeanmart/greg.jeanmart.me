---
title: Blockchain glossary
date: 2019-05-21
---

![](https://gateway.pinata.cloud/ipfs/QmXHu9jhPSs3Sw4nsN6WxTqajBZrtDK8ajifpSECNKibdc)


_A non-exhaustive and comprehensive Blockchain words glossary to help beginners to understand all the different concepts._

---------------------------------------------------------------------

### 51% attack

A 51% attack (or double spend attack) is a common Blockchain vulnerability when an attacker or a group of attacker control more than half of the computing power of the network and can consequently revise the transaction history or prevent new transactions.

---------------------------------------------------------------------

### Address

An address represents a unique identifier of an account on a Blockchain, it's a public information that can be shared with other actors of the network to send or receive assets. The address is derived from a private key.

Examples:
- Bitcoin: *1BoatSLRHtKNngkdXEeobR76b53LETtpyT*
- Ethereum: *0x123f681646d4a755815f9cb19e1acc8565a0c2ac*

---------------------------------------------------------------------

### Bitcoin

Bitcoin is the first Blockchain and cryptocurrency invented by Satoshi Nakamoto

---------------------------------------------------------------------

### Block

A Block represents a group of data containing transactions, the hash of the previous block (parent) and optionally other data.

---------------------------------------------------------------------

### Blockchain

A blockchain is a decentralized, peer2p shared ledger where transactions are immutably stored in blocks,  tamper-resistant and cryptographly verifiable.

---------------------------------------------------------------------

### Block explorer

A Block explorer is a tool to visualize transactions on the Blockchain. Such tools usually display more information like accounts balance, hash rate, coin supply, etc...

Examples:
- Ethereum Block explorer: https://etherscan.io/

---------------------------------------------------------------------

### Block reward

A block reward is a mechanism to encourage people to mine blocks. A miner who successfully solved the block gets a fixed reward. The value of the reward changes depending on the difficulty and other factors.

---------------------------------------------------------------------

### Consensus

A consensus mechanism (or algorithm) is a fault-tolerant mechanism used in blockchain system to achieve reliability in a network involving multiple unreliable nodes.

A consensus mechanism is required in most of the distributed system where each participant must come to an agreement. It doesn’t even matter whether participants of the system trust each other particularly or do not trust at all. All the same, they need to agree on certain principles of functioning that would be common for all of them.

Many consensus mechanisms exist like Proof of Work (PoW), Proof of Stake (PoS), Delegated Proof of Stake (dPoS), Proof of Authority (PoA).

---------------------------------------------------------------------

### Dapp

A Dapp (Decentralized APPlication) is an application built on top of decentralized technologies such as Blockchain. The backend logic is by design transparent, opensource and trusted and the data are immutable and tamperproof.

---------------------------------------------------------------------

### ERC20

ERC20 is a standard smart-contract protocol on the Ethereum blockchain used to issue a fungible token.

---------------------------------------------------------------------

### ERC721

ERC721, also called non-fungible token (NFT) or Collectible is a type of Ethereum standard smart-contract protocol to issue a token which is unique and non-fungible.

---------------------------------------------------------------------
### EVM

The Ethereum Virtual Machine (EVM) is the runtime environment for smart contracts in Ethereum.

---------------------------------------------------------------------

### Gas

Gas is the execution fee for every operation made on Ethereum. Its price is expressed in ether and it's decided by the miners, which can refuse to process transaction with less than a certain gas price.

When a block is mined, the miner is rewarded by all the gas included in each transaction included in the block.

---------------------------------------------------------------------

### Genesis Block

A genesis block is the first block of a blockchain. It is usually hardcoded in the protocol implementation itself and has a few rules (account initial balance, gas, etc...).

The Genesis block is the only block which doesn't have a parent.

---------------------------------------------------------------------

### Hardfork

A hardfork is a radical change in the blockchain protocol that makes invalid transactions valid, and vice versa. This requires all nodes to upgrade the protocol software to the latest version.
When both versions (pre-hardfork and post-hardfork) live together, it results to a split of the blockchain (fork) because the pre-hardfork nodes mine new blocks in the old protocol which are not compatible with the new blocks mined by the new protocol sotfware.

---------------------------------------------------------------------

### Hash

A cryptographic hash function is a function to compress an input data of a arbitrary length to a theoretically unique and fixed size data output called hash. This function is irreversible.

Hashing is used to index and retrieve items in a database because it is faster to find the item using the shorter hashed key than to find it using the original value.

---------------------------------------------------------------------

### Mining

In a PoW consensus based blockchain, mining is the process by which transactions are validated and added to the next block. Each miner validate the block and solve a cryptographic problem that requires a massive computing power, the first miner to solve the problem produces the block and earn a mining reward.

---------------------------------------------------------------------

### Multisig

Multisig (or multi-signature) is a digital signature scheme used when a transaction requires more than one signature to be valid. It's generally used to divide the responsibility for an action on the blockchain.

The multisig scheme requires two main information: a participants list and the number of required signature for a transaction to be valid.

---------------------------------------------------------------------

### Oracle

An Oracle is a software used to bridge data between the blockchain (smart contracts) and the real world.

A blockchain is deterministic state machine which means every time the a node validate the whole blockchain (every transaction), the same result is expected. If a smart contract was able to connect to a REST API for instance, the result could be different (status 200 OK or 500 INTERNAL SERVER ERROR if the service is down) which would invalidate the blockchain entirely.
Oracles save this restriction by receiving "requests" from a smart contracts (in the form of Events), execute the request (usually HTTP call) and save the response (at this given time) in the blockchain. If the result changes in the future, it won't be reflected in the smart contact except if it requests the data again to the Oracle.

---------------------------------------------------------------------

### PoA

Proof-of-Authority (PoA) is a consensus mechanism where block producers also known as validators are approved and trusted by the network. It's suitable for private blockchain network but can also be considered as a public blockchain that would require less decentralization (cf. POA network - sidechain). Validators are incentivize to well  behave because their identities is public and could undermine their reputations.

A blockhain running under PoA consensus has usually very fast transaction because of a shorter blocktimes and bigger blocks.

---------------------------------------------------------------------

### PoS

Proof-of-stake (PoS) is consensus mechanism where block producers also knows as stackers must show ownership and lock a certain amount of a digital currency. The new block producer is randomly picked up depending on his stake and can build and validate the new block. If the block producer produces a fraudulent block, he lose his stake.


---------------------------------------------------------------------

### PoW

Proof-of-Work (PoW) is a trustless and distributed consensus mechanism where anonymous block producers also known as miners uses computing power to calculate the next block and solve a variable difficulty cryptographic puzzle. The first miner to solve the problem produces the block and get a block reward.

---------------------------------------------------------------------

### Private/Public Key


Public-key cryptography, or asymmetric cryptography, is any cryptographic system that uses pairs of keys: public keys which may be disseminated widely, and private keys which are known only to the owner.

In blockchain, a private keys allow users to transact over the blockchain by signing and broadcasting transactions to the network. The signature can be recovered to validate the transaction and determine which account (identified by the public key) needs to be debited.

---------------------------------------------------------------------

### Smart contract

A smart contract is a self-executing, self-verifying and tamper resistant program with encoded business rules providing to network participants a software API to interact without third-party involved.

---------------------------------------------------------------------

### Solidity

Solidity is a programming language for writing smart contracts which run on Ethereum Virtual Machine.

---------------------------------------------------------------------

### Sybil attack

A Sybil attack is an attack where a malicious user controls multiple fake identities to influence the network with additional voting power for instance.

In peer2p network and Blockchain in particular, a Sybil attack usually takes place when the attacker takes over a majority of the node.

---------------------------------------------------------------------

### Transaction

A Blockchain transaction can be defined as a small unit of task that is stored, there records are stored in a block. A transaction usually includes all the necessary information to update and validate the state change like (from and to accounts, value, data, signature, nonce, etc.)

---------------------------------------------------------------------

### Wallet

A wallet is a interface that keeps one or more keypairs (Private/Public key) safe and allow users to manage their accounts (retrieve balance, send and sign transactions, interacts with contracts).

Different categories of wallet exists:

- Cold and Hot: A cold wallet stays most of the time offline and connects to the Internet only when required. While a hot wallet is online during all the session time.

- Third-party wallet or owned wallet: A third-party wallet is a wallet managed by someone else (Exchanges for instance), it usually offer a painless and easy access to the blockchain but at the cost of giving full control over your fund. If your own your wallet, your are responsible for it and no one than you can use it.

- Full-node or not: A wallet connected to a full node hosted by the owner will guarantee the legibility of the blockchain as full-nodes verify all transactions included in each block. If the wallet connects to a third-party node, wallet owners have to trust this provider.
