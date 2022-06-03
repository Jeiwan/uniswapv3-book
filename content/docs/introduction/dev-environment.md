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

1. An on-chain one, which is a set of smart contracts deployed on Ethereum.
1. An off-chain one, which is the front-end application that will interact with the smart contracts.

While the front-end application development is part of this book, it won't be our main focus. You probably have experience
in front-end or back-end development, so you have an idea of how these things work. Blockchains, on the other hand, are
different. Smart contracts development is similar to back-end development in some aspects, but it has its own nuances and
differences we need to know about. Let's begin by learning what is Ethereum and how it works.


## Quick Introduction to Ethereum

Ethereum is a blockchain that allows anyone to run applications on it. It might look like a cloud provider, but there's
one huge difference: you're not allowed to change once deployed applications. Smart contracts are immutable, you cannot
fix bugs or add new features without deploying a new contract. To better understand this limitation, let's see what Ethereum
is made of.

At the core of Ethereum (and any other blockchain) there's a database. The most valuable data in Ethereum's database is
*the state of accounts*. An account is an Ethereum address with associated data:

1. Balance: account's ether balance.
1. Code: bytecode of the smart contract deployed at this address.
1. Storage: space used by smart contracts to store data.
1. Nonce: a serial integer that's used to protect against replay attacks.

Ethereum's main task is to build and maintain this data in a secure way that doesn't allow unauthorized mutations of
data.

Ethereum is also a network, a network of computers that build and maintain this state independently of each other. The
main goal of the network is to **decentralize access to the database**: there must be no single authority that's allowed
to modify anything in the database unilaterally. This is achieved by a means of *consensus*, which is a set of rules all
the nodes in the network follow. If one party decides to abuse a rule, it'll be excluded from the network.

> Fun fact: blockchain can use MySQL! Nothing prevents this besides performance. In its turn, Ethereum uses 
[LevelDB](https://github.com/google/leveldb), a fast key-value database.

Every Ethereum node also runs EVM, Ethereum Virtual Machine. A virtual machine is a program that can run other programs,
and EVM is a program that runs smart contracts. Users interact with contracts through transactions: besides simply
sending ether, transactions can contain smart contract call data. Call data includes:

1. Encoded contract function name.
2. Function parameters.

In a sense, smart contracts are similar to APIs but instead of endpoints you call smart contract functions and you pass
function parameters. Similar to API backends, smart contracts execute programmed logic, which can optionally update
smart contract storage.

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

Forge makes smart contracts developer's life so much easier. With Forge, we don't need to run a local node and restart it
every time. Instead, Forge will run our smart contracts on its internal EVM, which is much faster and doesn't require
sending transactions and mining blocks.

Forge also allows us to write tests in Solidity! Before Forge, smart contract tests were written in JavaScript and this
required running a node, writing interactions with the node in JS, sending transactions, and mining blocks. Using Forge
also makes it easier to simulate blockchain state: we can easily fake our ether or token balance, execute contracts from
other addresses, deploy any contracts at any address, etc.

Anvil is a local Ethereum node which we'll use to deploy our smart contracts to. It uses automatic mining so we won't
need to wait for transactions, and it allows to mint any arbitrary amount of ether so we have enough of it for our
experiments.

### Ethers.js

[Ethers.js](https://github.com/ethers-io/ethers.js/) is an Ethereum wallet and a set of utilities implemented in JavaScript.
We can send transactions through Ethers.js, fetch data from blockchain, subscribe to events, etc. We'll use this library
in the front-end app to interact with our smart contracts

### MetaMask

[MetaMask](https://metamask.io/) is an Ethereum wallet in your browser. It's a browser extension that allows to create
and securely store Ethereum private keys. It's the main Ethereum wallet application used by millions of users. We'll need
a wallet with test ether to be able to pay for transactions.

### React

[React](https://reactjs.org/) is a well-known JavaScript library for building front-end applications. You don't need to
know React, I'll provide a template application.