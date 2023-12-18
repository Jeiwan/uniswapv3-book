# NFT Positions

This is the cherry on the cake of this book. In this milestone, we're going to learn how Uniswap contracts can be extended and integrated into third-party protocols. This possibility is a direct consequence of having core contracts with only crucial functions, which allows for integration into other contracts without the need to add new features to core contracts.

A bonus feature of Uniswap V3 was the ability to turn liquidity positions into NFT tokens. Here's an example of one such token:

<p align="center">
<img src="images/nft_example.png" alt="Uniswap V3 NFT example"/>
</p>

It shows token symbols, pool fees, position ID, lower and upper ticks, token addresses, and the segment of the curve where the position is provided.

> You can see all Uniswap V3 NFT positions in [this OpenSea collection](https://opensea.io/collection/uniswap-v3-positions).

In this milestone, we're going to add NFT tokenization of liquidity positions!

Let's go!

> You'll find the complete code of this chapter in [this Github branch](https://github.com/Jeiwan/uniswapv3-code/tree/milestone_6).
>
> This milestone introduces a lot of code changes in existing contracts. [Here you can see all changes since the last milestone](https://github.com/Jeiwan/uniswapv3-code/compare/milestone_5...milestone_6)

> If you have any questions feel free to ask them in [the GitHub Discussion of this milestone](https://github.com/Jeiwan/uniswapv3-book/discussions/categories/milestone-6-nft-positions)!