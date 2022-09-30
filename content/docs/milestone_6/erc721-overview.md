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

What Uniswap V3 liquidity positions and NFTs have in common is this non-fungibility: NFTS and liquidity positions are not
interchangeable and are identified by unique IDs. It's this similarity that will allow us to merge the two concepts.

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

Such JSON file defines: the name of a token, the description of a collection, the link to the image of a token, properties of a
token.

Alternatively, we may store JSON metadata and token images on-chain. This is very expensive of course (saving data on-chain
is the most expensive operation in Ethereum), but we can make it cheaper if we store templates. All tokens within a collection
have similar metadata (mostly identical but image links and properties are different for each token) and visuals. For the 
latter, we can use SVG, which is an HTML-like format, and HTML is a good templating language.

When storing JSON metadata and SVG on-chain, the `tokenURI` function, instead of returning a link, would return JSON
metadata directly, using the [data URI scheme](https://en.wikipedia.org/wiki/Data_URI_scheme#Syntax) to encode it. SVG
images would also be inlined, it won't be necessary making external requests to download token metadata and image.