# Uniswap V3 Development Book

<p align="center">
<img src="/src/images/cover.png" alt="Uniswap V3 Development Book cover" width="360"/>
</p>


<p align="center">
👉&nbsp;<a href="https://uniswapv3book.com/">READ ONLINE</a>&nbsp;&nbsp;|&nbsp;&nbsp;<a href="https://uniswapv3book.com/print.html">PRINT OR SAVE AS PDF</a>&nbsp;👈
</p>

This book will teach how to develop an advanced decentralized application! Specifically, we'll be building a clone of
[Uniswap V3](https://uniswap.org/), which is a decentralized exchange.

## Why Uniswap?
- It implements a very simple mathematical concept, `x * y = k`, which still makes it very powerful.
- It's an advanced application that has a thick layer of engineering on top of the simple formula.
- It's permissionless and battle-tested. Learning from an application that's been running in production for
several years and handling billions of dollars will make you a better developer.

## What we'll build

![Front-end application screenshot](/screenshot.png)

We'll build a full clone of Uniswap V3. It **won't be an exact copy** and it **won't be production-ready** because we'll
do something in our own way and we'll **definitely** introduce multiple bugs. So, don't deploy this to the mainnet!

While our focus will primarily be on smart contracts, we'll also build a front-end application as a side hustle. 🙂
I'm not a front-end developer and I cannot make a front-end application better than you, but I can show you how a
decentralized exchange can be integrated into a front-end application.

The full code of what we'll build is stored in a separate repository:

https://github.com/Jeiwan/uniswapv3-code

You can read this book at:

https://uniswapv3book.com/

### Questions?

Each milestone has its own section in [the GitHub Discussions](https://github.com/Jeiwan/uniswapv3-book/discussions).
Don't hesitate to ask questions about anything that's not clear in the book!

## Table of Contents

- Milestone 0. Introduction
  1. Introduction to markets
  1. Constant Function Market Makers
  1. Uniswap V3
  1. Development Environment
  1. What We'll Build
- Milestone 1. First Swap
  1. Introduction
  1. Calculating Liquidity
  1. Providing Liquidity
  1. First Swap
  1. Manager Contract
  1. Deployment
  1. User Interface
- Milestone 2. Second Swap
  1. Introduction
  1. Output Amount Calculation
  1. Math in Solidity
  1. Tick Bitmap Index
  1. Generalize Minting
  1. Generalize Swapping
  1. Quoter Contract
  1. User Interface
- Milestone 3. Cross-tick Swaps
  1. Introduction
  1. Different Price Ranges
  1. Cross-Tick Swaps
  1. Slippage Protection
  1. Liquidity Calculation
  1. A Little Bit More on Fixed-point Numbers
  1. Flash Loans
  1. User Interface

- Milestone 4. Multi-pool Swaps
  1. Introduction
  1. Factory Contract
  1. Swap Path
  1. Multi-pool Swaps
  1. User Interface
  1. Tick Rounding
- Milestone 5. Fees and Price Oracle
  1. Introduction
  1. Swap Fees
  1. Flash Loan Fees
  1. Protocol Fees
  1. Price Oracle
  1. User Interface
- Milestone 6: NFT positions
  1. Introduction
  1. ERC721 Overview
  1. NFT Manager
  1. NFT Renderer

## Running locally

To run the book locally:
1. Install [Hugo](https://gohugo.io/).
1. Clone the repo and the theme submodule:
    ```shell
    $ git clone https://github.com/Jeiwan/uniswapv3-book --recurse-submodules
    $ cd uniswapv3-book
    ```
1. Run:
    ```shell
    $ hugo server -D
    ```
1. Visit http://localhost:1313/ (or whatever URL the previous command outputs!)