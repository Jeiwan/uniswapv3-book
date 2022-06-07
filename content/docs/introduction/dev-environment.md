---
title: "Development Environment"
weight: 4
# bookFlatSection: false
# bookToc: true
# bookHidden: false
# bookCollapseSection: false
# bookComments: false
# bookSearchExclude: false
---

# Development environment

We're going to build two applications:

1. An on-chain one–a set of smart contracts deployed on Ethereum.
1. An off-chain one–a front-end application that will interact with the smart contracts.

While the front-end application development is part of this book, it won't be our main focus. We will build it solely to
demonstrate how smart contracts are integrated with front-end applications. You won't need to build it from scratch,
I'll provide a template.

## Quick Introduction to Ethereum

Ethereum is a blockchain that allows anyone to run applications on it. It might look like a cloud provider, but there are
multiple differences:
1. You don't pay for hosting your application. But you pay for deployment.
1. Your application will be small and it will be able to interact only with other contracts and the internal blockchain
storage.
1. Your application is immutable. That is: you won't be able to modify it after it's deployed.

To better understand these moments, let's see what Ethereum is made of.

At the core of Ethereum (and any other blockchain) is a database. The most valuable data in Ethereum's database is
*the state of accounts*. An account is an Ethereum address with associated data:

1. Balance: account's ether balance.
1. Code: bytecode of the smart contract deployed at this address.
1. Storage: space used by smart contracts to store data.
1. Nonce: a serial integer that's used to protect against replay attacks.

Ethereum's main job is building and maintaining this data in a secure way that doesn't allow unauthorized access.

Ethereum is also a network, a network of computers that build and maintain the state independently of each other. The
main goal of the network is to **decentralize access to the database**: there must be no single authority that's allowed
to modify anything in the database unilaterally. This is achieved by a means of *consensus*, which is a set of rules all
the nodes in the network follow. If one party decides to abuse a rule, it'll be excluded from the network.

> Fun fact: blockchain can use MySQL! Nothing prevents this besides performance. In its turn, Ethereum uses 
[LevelDB](https://github.com/google/leveldb), a fast key-value database.

Every Ethereum node also runs EVM, Ethereum Virtual Machine. A virtual machine is a program that can run other programs,
and EVM is a program that executes smart contracts. Users interact with contracts through transactions: besides simply
sending ether, transactions can contain smart contract call data. It includes:

1. An encoded contract function name.
2. Function parameters.

Transactions are packed in blocks and blocks then mined by miners. Each participant of the network can validate any
transaction and any block.

In a sense, smart contracts are similar to JSON APIs but instead of endpoints you call smart contract functions and you
pass function parameters. Similar to API backends, smart contracts execute programmed logic, which can optionally modify
smart contract storage. Unlike JSON API, you need to send a transaction to mutate blockchain state, and you'll need to
pay for each transaction you're sending.

Finally, Ethereum nodes expose a JSON-RPC API. Through this API, we can get account balance, estimate gas costs, get blocks
and transactions, send transactions, and execute contract calls without sending transactions (this is used to read data
from smart contracts). [Here](https://eth.wiki/json-rpc/API) you can find the full list of available endpoints.

## Local Development Environment

We're going to build smart contracts and run them on Ethereum, which means we need a node. And this is what smart
contracts development looked like until recently. Today, we don't need to run a node, which makes development much
faster and allows us to iterate quicker.

Let's review the tools we're going to use.

### Foundry

[Foundry](https://github.com/foundry-rs/foundry) is a set of tools for Ethereum applications development. Specifically,
we're going to use:
1. [Forge](https://github.com/foundry-rs/foundry/tree/master/forge), a testing framework for Solidity.
1. [Anvil](https://github.com/foundry-rs/foundry/tree/master/anvil), a local Ethereum node designed for development with
Forge.

[TODO: maybe Cast?]

Forge makes smart contracts developer's life so much easier. With Forge, we don't need to run a local node to test
contracts. Instead, Forge will run our smart contracts on its internal EVM, which is much faster and doesn't require
sending transactions and mining blocks.

Forge allows us to write tests in Solidity! Before Forge, smart contract tests were written in JavaScript and this
required running a node, writing interactions with the node in JS, sending transactions, and mining blocks. Forge also
makes it easier to simulate blockchain state: we can easily fake our ether or token balance, execute contracts from other
addresses, deploy any contracts at any address, etc.

However, we'll still need a local node to deploy our contract to. For that, we'll use Anvil.

### Ethers.js

[Ethers.js](https://github.com/ethers-io/ethers.js/) is a set of Ethereum utilities written in JavaScript. This is one
of the two (the other one is [web3.js](https://github.com/ChainSafe/web3.js)) JavaScript libraries that are used in
decentralized applications development. These libraries allow us to interact with an Ethereum node via the JSON-API, and
they come with multiple utility functions that make developer's life easier.

### MetaMask

[MetaMask](https://metamask.io/) is an Ethereum wallet in your browser. It's a browser extension that creates and securely
stores Ethereum private keys. MetaMask is the main Ethereum wallet application used by millions of users. We'll use it
to sign transactions that we'll send to our local node.

### React

[React](https://reactjs.org/) is a well-known JavaScript library for building front-end applications. You don't need to
know React, I'll provide a template application.