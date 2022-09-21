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

# Cross-tick Swaps

We have made a great progress so far and our Uniswap V3 implementation is quote close to the original one! However, our
implementation only supports swaps within a price range–and this is what we're going to improve in this milestone.

In this milestone, we'll:
1. update `mint` function to provide liquidity in different price ranges;
1. update `swap` function to cross price ranges when there's not enough liquidity in the current price range;
1. learn how to calculate liquidity in smart contracts;
1. implement slippage protection in `mint` and `swap` functions;
1. update the UI application to allow to add liquidity at different price ranges;
1. learn a little bit more about fixed-point numbers.

In this milestone, we'll complete swapping, the core functionality of Uniswap!

Let's begin!

> You'll find the complete code of this chapter in [this Github branch](https://github.com/Jeiwan/uniswapv3-code/tree/milestone_3).
>
> This milestone introduces a lot of code changes in existing contracts. [Here you can see all changes since the last milestone](https://github.com/Jeiwan/uniswapv3-code/compare/milestone_2...milestone_3)