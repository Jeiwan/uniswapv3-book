---
title: "Deployment"
weight: 6
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


## Choosing Local Blockchain Network

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

There are multiple solutions as of today:
1. [Ganache](https://trufflesuite.com/ganache/) from Truffle Suite.
1. [Hardhat](https://hardhat.org/), which is a development environment that includes a local node besides other useful
things.
1. [Anvil](https://github.com/foundry-rs/foundry/tree/master/anvil) from Foundry.

All of these are viable solutions and each of them will satisfy our needs. Having said that, projects have been slowly
migrating from Ganache (which is the oldest of the solutions) to Hardhat (which seems to be the most widely used these
days), and now there's the new kid on the block: Foundry. Foundry is also the only of these solutions that uses Solidity
for writing tests (the others use JavaScript). Moreover, Foundry also allows to write deployment scripts in Solidity.
Thus, since we've decided to use Solidity everywhere, we'll use Anvil to run a local development blockchain, and we'll
write deployment scripts in Solidity.