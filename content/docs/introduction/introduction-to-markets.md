---
title: "Introduction to Markets"
weight: 1
# bookFlatSection: false
# bookToc: false
# bookHidden: false
# bookCollapseSection: false
# bookComments: false
# bookSearchExclude: false
---


# Introduction to markets

## How centralized exchanges work
In this book, we'll build a decentralized exchange (DEX) that will run on Ethereum. There're multiple approaches to how an
exchange can be designed. All centralized exchanges have *an order book* at their core. An order book is just a journal that
stores all sell and buy orders that traders want to make. Each order in this book contains a price the order must be
executed at and the amount that must be bought or sold.

![Order book example](/images/milestone_0/orderbook.png)

For trading to happen, there must exist *liquidity*, which is simply the availability of assets on a market. If you
want to buy a wardrobe but no one is selling one, there's no liquidity. If you want to sell a wardrobe but no one wants
to buy it, there's liquidity but no buyers. If there's no liquidity, there's nothing to buy or sell.

On centralized exchanges, the order book is where liquidity is accumulated. If someone places a sell order, they provide
liquidity to the market. If someone places a buy order, they expected the market to have liquidity, otherwise, no trade
is possible.

When there's no liquidity, but markets are still interested in trades, *market makers* come into play. A market maker is
a firm or an individual who provides liquidity to markets, that is someone who has a lot of money and who buys different
assets to sell them on exchanges. For this job market makers are paid by exchanges. **Market makers make money on
providing liquidity to exchanges**.

## How decentralized exchanges work

Don't be surprised, decentralized exchanges also need liquidity. And they also need someone who provides it to traders
of a wide variety of assets. However, this process cannot be handled in a centralized way. **A decentralized solution
must be found.** There are multiple decentralized solutions and some of them are implemented differently. Our focus will
be on how Uniswap solves this problem.

## Automated Market Makers

[The evolution of on-chain markets](https://bennyattar.substack.com/p/the-evolution-of-amms) brought us to the idea of
Automated Market Makers (AMM). As the name implies, this algorithm works exactly like market makers but in an automated
way. Moreover, it's decentralized and permissionless, that is:
- it's not governed by a single entity;
- all assets are not stored in one place;
- anyone can use it from anywhere.

### What is an AMM?

An AMM is a set of smart contracts that define how liquidity is managed. Each trading pair (e.g. ETH/USDC) is a separate
contract that stores both ETH and USDC and that's programmed to mediate trades: exchanging ETH for USDC and vice versa.

The core idea is **pooling**: each contract is a *pool* that stores liquidity let's different users (including other
smart contract) to trade in a permissioned way. There are two roles, *liquidity providers* and traders, and these roles
interact with each other through pools of liquidity, and the way they can interact with pools is programmed and immutable.

![Automated Market Maker simplified](/images/milestone_0/amm_simplified.png)

What makes this approach different from centralized exchanges is that **the smart contracts are fully automated and not
managed by anyone**. There are no managers, admins, privileged users, etc. There are only liquidity providers and traders
(they can be the same people), and all the algorithms are programmed, immutable, and public.

Let's now look closer at how Uniswap implements an AMM.

> Please note that I use *pool* and *pair* terms interchangeably throughout the book because a Uniswap pool is a pair
of two tokens.

> If you have any questions feel free asking them in [the GitHub Discussion of this milestone](https://github.com/Jeiwan/uniswapv3-book/discussions/categories/milestone-0-introduction)!
