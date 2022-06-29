---
title: "Deployment"
weight: 5
# bookFlatSection: false
# bookToc: true
# bookHidden: false
# bookCollapseSection: false
# bookComments: false
# bookSearchExclude: false
---

# Deployment

Alright, our contract is done. Now, let's see how we can deploy it to a local Ethereum network so we could use it from
a front-end app later on.


## Running Local Ethereum Network

Smart contracts development requires running a local network, where you deploy your contracts during development and
testing. This is what we want from such a network:
1. Real blockchain. It must be a real Ethereum network, not an emulation. We want to be sure that our contract will work
in the local network exactly as it would in the mainnet.
1. Speed. We want our transactions to be minted immediately, so we could iterate quickly.
1. Ether. To pay transaction fees, we need some ether, and we want the local network to allow us to generate any amount
of ether.
1. Cheat codes. Besides providing the standard API, we want a local network to allow us to do more. For example, we want
to be able to deploy contracts at any address, execute transactions from any address (impersonate other address), change
contract state directly, etc.