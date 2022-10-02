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

# Multi-pool Swaps

After implementing cross-tick swaps, we've got really close to real Uniswap V3 swaps. One significant limitation of our
implementation is that it allows only swaps within a pool–if there's no pool for a pair of tokens, then swapping between
these tokens is not possible. This is not so in Uniswap since it allows multi-pool swaps. In this chapter, we're going
to add multi-pool swaps to our implementation.

Here's the plan:

1. first, we'll learn about and implement Factory contract;
1. then, we'll see how chained or multi-pool swaps work and implement Path library;
1. then, we'll update the front-end app to support multi-pool swaps;
1. we'll implement a basic router that finds a path between two tokens;
1. along the way, we'll also learn about tick spacing which is a way of optimizing swaps.


After finishing this chapter, our implementation will be able to handle multi-pool swaps, for example, swapping WBTC for
WETH via different stablecoins: WETH → USDC → USDT → WBTC.

Let's begin!


> You'll find the complete code of this chapter in [this Github branch](https://github.com/Jeiwan/uniswapv3-code/tree/milestone_4).
>
> This milestone introduces a lot of code changes in existing contracts. [Here you can see all changes since the last milestone](https://github.com/Jeiwan/uniswapv3-code/compare/milestone_3...milestone_4)

> If you have any questions feel free asking them in [the GitHub Discussion of this milestone](https://github.com/Jeiwan/uniswapv3-book/discussions/categories/milestone-4-multi-pool-swaps)!