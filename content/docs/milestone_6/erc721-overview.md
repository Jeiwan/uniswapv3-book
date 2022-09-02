---
title: "ERC721 Overview"
weight: 2
# bookFlatSection: false
# bookToc: true
# bookHidden: false
# bookCollapseSection: false
# bookComments: false
# bookSearchExclude: false
---

# ERC721 Overview

Let's begin with an overview of [EIP-721](https://eips.ethereum.org/EIPS/eip-721), the standard that defines NFT
contracts.

ERC721 is a variant of ERC20. The main difference between them is that ERC721 tokens are *non-fungible*, that is: one
token is not identical to another. To distinguish ERC721 tokens, each of them has a unique ID, which is almost always
the counter at which a token was minted. ERC721 tokens also have an extended concept of ownership: owner of each token
is tracked and stored in the contract. This means that only distinct tokens, identified by token IDs, can be transferred
(or approved for transfer).

Unique identity is what NFT tokens and Uniswap V3 liquidity positions have in common: each NFT token is identified by its
unique ID and each liquidity position is identified by an ID (which is computed from position owner address and the price
range). It's this similarity that will allow us to integrate the two concepts.

The biggest difference between ERC20 and ERC721 is the `tokenURI` function in the latter. NFT tokens, which are
implemented as ERC721 smart contracts, have linked assets that are stored externally, not on blockchain. To link token IDs to
images (or sounds, or anything else) stored outside of blockchain, ERC721 defines the `tokenURI` function. The function
is expected to return a link to a JSON file that defines NFT token metadata, e.g.:
```json
{
    "name": "Thor's hammer",
    "description": "Mjölnir, the legendary hammer of the Norse god of thunder.",
    "image": "https://game.example/item-id-8u5h2m.png",
    "strength": 20
}
```
(This example is taken from the [ERC721 documentation on OpenZeppelin](https://docs.openzeppelin.com/contracts/4.x/erc721))

Such JSON file defines: name of a token, description of a collection, link to the image of a token, properties of a
token.

Optionally, we can store JSON metadata files and token images on-chain. We don't actually need to store full JSON files
for each of the token in a collection because we can simply generate them on-the-fly (when `tokenURI` is called) and
fill the fields with token-specific data. The same is true if we use SVG for token images: we can store an SVG template
on-chain and generate token-specific SVG when `tokenURI` is called. To store JSON and SVG on-chain, we can use the [data URI scheme](https://en.wikipedia.org/wiki/Data_URI_scheme#Syntax)
(and this is what we'll do!).