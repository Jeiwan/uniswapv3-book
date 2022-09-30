---
title: "Introduction"
weight: 1
# bookFlatSection: false
# bookToc: true
# bookHidden: false
# bookCollapseSection: false
# bookComments: false
# bookSearchExclude: false
---

{{< katex display >}} {{</ katex >}}

# NFT Positions

This is the cherry on the cake of this book. In this milestone, we're going to learn how Uniswap contract can be extended
and integrated into third-party protocols. This possibility is a direct consequence of having core contracts with only
crucial functions, which allows to integrate them into other contracts without the need of adding new features to core 
contracts.

A bonus feature of Uniswap V3 was the ability to turn liquidity positions into NFT tokens. Here's an example of one
such NFT tokens:

![Uniswap V3 NFT example](/images/milestone_6_nft_example.png)

It shows token symbols, pool fee, position ID, lower and upper ticks, token addresses, and the segment of the curve the
position is provided at.

> You can see all Uniswap V3 NFT positions in [this OpenSea collection](https://opensea.io/collection/uniswap-v3-positions).

In this milestone, we're going to add NFT-tokenization of liquidity positions!

Let's go!