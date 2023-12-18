# Fees and Price Oracle

In this milestone, we're going to add two new features to our Uniswap implementation. They share one similarity: they work on top of what we have already built–that's why we've delayed them until this milestone. However, they're not equally important.

We're going to add swap fees and a price oracle:
- Swap fees is a crucial mechanism of the DEX design we're implementing. They're the glue that makes things stick together. Swap fees incentivize liquidity providers to provide liquidity, and no trades are possible without liquidity, as we have already learned.
- A price oracle, on the other hand, is an optional utility function of a DEX. A DEX, while conducting trades, can also function as a price oracle–that is, provide token prices to other services. This doesn't affect actual swaps but provides a useful service to other on-chain applications.

Alright, let's get building!

> You'll find the complete code of this chapter in [this Github branch](https://github.com/Jeiwan/uniswapv3-code/tree/milestone_5).
>
> This milestone introduces a lot of code changes in existing contracts. [Here you can see all changes since the last milestone](https://github.com/Jeiwan/uniswapv3-code/compare/milestone_4...milestone_5)

> If you have any questions feel free to ask them in [the GitHub Discussion of this milestone](https://github.com/Jeiwan/uniswapv3-book/discussions/categories/milestone-5-fees-and-price-oracle)!