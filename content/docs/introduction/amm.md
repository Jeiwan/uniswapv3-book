---
title: "Introduction to markets"
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
In this book we'll build a decentralized exchange (DEX) that will run on Ethereum. There're multiple approaches to how an
exchange can be designed. All centralized exchanges have *an order book* at their core. Order book is just a journal that
stores all sell or buy orders that traders what to make. Each order in this book contains: a price the order must be
executed at and the amount that must be bought or sold.

[TODO: illustration]

For trading to happen, there must exist *liquidity*, which is simply availability of assets on a market. If you
want to buy a wardrobe but no one is selling one, there's no liquidity. If you want to sell a wardrobe but no one wants
to buy it, there's liquidity but no buyers. If there's no liquidity, there's nothing to buy or sell.

On centralized exchanges, order books is where liquidity is accumulated. If someone places a sell order, they provide
liquidity to the market. If someone places a buy order, they expected the market to have liquidity, otherwise no trade
is possible.

Since liquidity is not always available, but markets are still interested in trades, entities called *market makers* were
established. Market maker is a firm or an individual who provides liquidity to markets, that is: some one who has a lot
of money and who buys different assets to sell them on exchanges. For this job, market makers are paid by exchanges.

## How decentralized exchanges work

Don't be surprized, decentralized exchanges also need liquidity. And they also need someone who provides it to traders
of a wide variety of assets. However, this process cannot be handled in a centralized way. A decentralized solution
must exist. There are multiple of them, and one of them turned to be the simplest and satisfying all requirements.

## Automated Market Makers

The evolution of on-chain markets brought us to the idea of Automated Market Makers (AMM). As the name implies, this algorithm
works exactly like market makers but in an automated way. And in decentralized, which was the main requirement.

### What is an AMM?

The core idea is **pooling**: different and not connected groups of people are incentivized to put their assets (tokens)
into *pools*, which are smart contracts. Anyone else is able to use these pool contracts to trade, thanks to liquidity
provided by the first group.

[TODO: illustration]

What makes this approach different from centralized exchanges is that **the smart contracts are fully automated and not
managed by anyone**. There are no managers, admins, privileged users, etc. There are only **liquidity providers** and 
**traders**, and all the algorithms are programmed and immutable.