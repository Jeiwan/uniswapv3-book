# 🚧 Uniswap V3 Development Book 🚧

## Table of Contents

### Introduction
1. ~~Introduction, overview of the course~~
1. ~~Automated Market Makers (overview, V2, theory, math, visualizations, curve, Dan's desmos)~~
1. ~~Uniswap V3 (overview of V3, concentrated liquidity, L, sqrt(P))~~
1. ~~Tools (quick intro to blockchains, Foundry, Hardhat, testing, local node; Metamask, React, Ethers.js)~~

### Milestone 1: First swap
1. ~~Liquidity provision (minting)~~
1. ~~First swap (Pool contract, 0-to-1, hardcoded ticks and sqrt(P))~~
1. ~~Front-end app (basic React app, Metamask, integration with the contract, swaps through the UI)~~

### Milestone 2: Second swap
1. ~~Math libraries~~
1. ~~Calculated ticks and sqrt(P)~~
1. ~~Second swap (1-to-0)~~
1. ~~Quoter contract~~
1. ~~Updated front end~~

### Milestone 3: Cross-tick swaps
1. ~~Cross-tick swaps, theory~~
1. ~~Adding liquidity to different price ranges~~
1. ~~Cross-tick swaps, implementation~~
1. ~~Slippage protection~~
1. ~~Update 'Add Liquidity' interface~~

### Milestone 4: Factory contract
1. Factory, theory
1. Deploying via Factory
1. Using Factory everywhere (Manager, Quoter, UI)
1. Chained swaps
1. Auto Router (simple implementation, DFS maybe).
1. Front end: allow to create pairs via the UI

### Milestone 5: Fees and price oracle
1. Trading fees, theory
1. Collecting fees
1. Withdrawing fees
1. Removing liquidity (burn)
1. Protocol fees
1. Overview of the price oracle in Uniswap V3
1. Implementation in Pool contract
1. Front end: allow to add/remove liquidity through the UI
1. Front end: add a chart with historical prices obtained via the price oracle
(Oracle CLI tool as a helper)

### Milestone 6: Extras
1. Overview and implementation of flash loans
1. Implementing NFT position manager (wrapping positions in NFTs)
1. Overview of NFTSVG contract (as a home work maybe)
1. Tick math implementation. Explanation of the math in [TickMath](https://github.com/Uniswap/v3-core/blob/main/contracts/libraries/TickMath.sol) contract.