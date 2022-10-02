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

# Second Swap

Alright, this is where it gets real. So far, our implementation has been looking too synthetic and static. We have
calculated and hard coded all the amounts to make the learning curve less steep, and now we're ready to make it dynamic.
We're going to implement the second swap, that is a swap in the opposite direction: sell ETH to buy USDC. To do this,
we're going to improve our smart contracts significantly:
1. We need to implement math calculations in Solidity. However, since implementing math in Solidity is tricky due to
Solidity supporting only integer division, we'll use third-party libraries.
1. We'll need to let users choose swap direction, and the pool contract will need to support swapping in both directions.
We'll improve the contract and will bring it closer to multi-range swaps, which we'll implement in the next milestone.
1. Finally, we'll update the UI to support swaps in both directions AND output amount calculation! This will require us
implementing another contract, Quoter.

In the end of this milestone, we'll have an app that works almost like a real DEX!

Let's begin!

> You'll find the complete code of this chapter in [this Github branch](https://github.com/Jeiwan/uniswapv3-code/tree/milestone_2).
>
> This milestone introduces a lot of code changes in existing contracts. [Here you can see all changes since the last milestone](https://github.com/Jeiwan/uniswapv3-code/compare/milestone_1...milestone_2)

> If you have any questions feel free asking them in [the GitHub Discussion of this milestone](https://github.com/Jeiwan/uniswapv3-book/discussions/categories/milestone-2-second-swap)!