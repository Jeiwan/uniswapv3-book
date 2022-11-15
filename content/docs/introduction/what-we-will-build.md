---
title: "What We'll Build"
weight: 5
# bookFlatSection: false
# bookToc: true
# bookHidden: false
# bookCollapseSection: false
# bookComments: false
# bookSearchExclude: false
---

# What We'll Build

The goal of the book is to build a clone of Uniswap V3. However, we won't build an exact copy. The main reason is that
Uniswap is a big project with many nuances and auxiliary mechanics–breaking down all of them would bloat the book and make
it harder for readers to finish it. Instead, we'll build the core of Uniswap, its hardest and most important mechanisms.
This includes liquidity management, swapping, fees, a periphery contract, a quoting contract, and an NFT contract. After
that, I'm sure, you'll be able to read the original source code of Uniswap V3 and understand all the mechanics that were
left outside of the scope of this book.


## Smart Contracts

After finishing the book, you'll have these contracts implemented:
1. `UniswapV3Pool`–the core pool contract that implements liquidity management and swapping. This contract is very close
to [the original one](https://github.com/Uniswap/v3-core/blob/main/contracts/UniswapV3Pool.sol), however, some implementation
details are different and something is missed for simplicity. For example, our implementation will only handle "exact input"
swaps, that is swaps with known input amounts. The original implementation also supports swaps with known *output* amounts
(i.e. when you want to buy a certain amount of tokens).
1. `UniswapV3Factory`–the registry contract that deploys new pools and keeps a record of all deployed pools. This one is
mostly identical to [the original one](https://github.com/Uniswap/v3-core/blob/main/contracts/UniswapV3Factory.sol) besides
the ability to change owner and fees.
1. `UniswapV3Manager`–a periphery contract that makes it easier to interact with the pool contract. This is a very simplified
implementation of [SwapRouter](https://github.com/Uniswap/v3-periphery/blob/main/contracts/SwapRouter.sol). Again, as you
can see, I don't distinguish "exact input" and "exact output" swaps and implement only the former ones.
1. `UniswapV3Quoter` is a cool contract that allows calculating swap prices on-chain. This is a minimal copy of both [Quoter](https://github.com/Uniswap/v3-periphery/blob/main/contracts/lens/Quoter.sol)
and [QuoterV2](https://github.com/Uniswap/v3-periphery/blob/main/contracts/lens/QuoterV2.sol). Again. only "exact input"
swaps are supported.
1. `UniswapV3NFTManager` allows turning liquidity positions into NFTs. This is a simplified implementation of [NonfungiblePositionManager](https://github.com/Uniswap/v3-periphery/blob/main/contracts/NonfungiblePositionManager.sol).


## Front-end Application

For this book, I also built a simplified clone of [the Uniswap UI](https://app.uniswap.org/). This is a very dumb clone,
and my React and front-end skills are very poor, but it demonstrates how a front-end application can interact with smart
contracts using Ethers.js and MetaMask.