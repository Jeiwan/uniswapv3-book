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

> You'll find the complete code of this chapter in [this Github branch](https://github.com/Jeiwan/uniswapv3-code/tree/milestone_1).

# First Swap

In this milestone, we'll build a pool contract that can receive liquidity from users and make swaps within a price range.
To keep it as simple as possible, we'll provide liquidity only in one price range and we'll make a swap only in one
direction. Also, we'll calculate all the required math manually to get better intuition before starting using mathematical libs.

Let's model the situation we'll build:
1. There will be an ETH-USDC pool contract. ETH will be the $x$ reserve; USDC will be the $y$ reserve.
1. We'll set the current price to 5000 USDC per 1 ETH.
1. The range we'll provide liquidity is 4500-5500 USDC per 1 ETH.
1. We'll buy some ETH from the pool. At this point, since we have only one price range, we want the price of the trade
to stay within the price range.

Visually, this model looks like this:

[TODO: graph the model]

Before getting to code, let's figure out the math and calculate all the parameters of the model.