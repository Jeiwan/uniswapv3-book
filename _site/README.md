# 🚧 Uniswap V3 Development Book 🚧

![Front-end application screenshot](/screenshot.png)

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
1. ~~Flash loans~~
1. ~~Update 'Add Liquidity' interface~~

### Milestone 4: Multi-pool Swaps
1. ~~Factory, theory~~
1. ~~Deploying via Factory~~
1. ~~Using Factory everywhere (Manager, Quoter, UI)~~
1. ~~Path library~~
1. ~~Chained swaps~~
1. ~~Auto Router (simple implementation, DFS maybe).~~
1. ~~Front end: allow to create pairs via the UI~~
1. ~~Tick Rounding~~

### Milestone 5: Fees and price oracle
1. ~~Trading fees, theory~~
1. ~~Collecting fees~~
1. ~~Withdrawing fees~~
1. ~~Removing liquidity (burn)~~
1. ~~Protocol fees~~
1. ~~Overview of the price oracle in Uniswap V3~~
1. ~~Implementation in Pool contract~~
1. ~~Front end: allow to add/remove liquidity through the UI~~

### Milestone 6: NFT-positions
1. ~~NFT-position manager contract~~
1. ~~Basic NFTSVG contract~~
1. ~~Front end: add an endpoint to render NFT-positions~~

### TODO
1. MOVE TO MILESTONE 2: Tick math implementation. Explanation of the math in [TickMath](https://github.com/Uniswap/v3-core/blob/main/contracts/libraries/TickMath.sol) contract.