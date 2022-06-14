---
title: "First Swap"
weight: 1
# bookFlatSection: false
# bookToc: false
# bookHidden: false
# bookCollapseSection: false
# bookComments: false
# bookSearchExclude: false
---

{{< katex display >}} {{</ katex >}}

> You'll find the complete code of this chapter in [this Github branch](https://github.com/Jeiwan/uniswapv3-code/tree/milestone_1).

# First Swap

Enough of theory, let's start coding!

Create a new folder (mine is called `uniswapv3-code`), and run `forge init --vscode` in it–this will initialize a Forge
project. The `--vscode` flag tells Forge to configure the Solidity extension for Forge projects.

Next, remove the default contract and its test:
- `script/Contract.s.sol`
- `src/Contract.sol`
- `test/Contract.t.sol`

And that's it! Let's create our first contract!

## Pool Contract

As you've learned from the introduction, Uniswap deploys multiple Pool contracts, each of which is an exchange market of
a pair of tokens. Uniswap groups all its contract into two categories:

- core contracts, and
- periphery contracts.

Core contracts are, as the name implies, the contracts that implement core logic. These are minimal, user-*un*friendly,
low-level contracts. Their purpose is to do one thing. In Uniswap V3, there are 2 such contracts:
1. Pool contract, which implements the core logic of a decentralized exchange.
1. Factory contract, which serves as a registry of Pool contracts and a contract that makes deployment of pools easier.

We'll begin with the pool contract. Create `src/UniswapV3Pool.sol`:

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.14;

contract UniswapV3Pool {}
```

Let's think about what data the contract will store:
1. Since every pool contract is an exchange market of two tokens, we need to track the two token addresses. And these
addresses will be static, set once and forever during contract initialization.
1. Each pool contract is a set of liquidity positions, a data structure to manage positions identified by: liquidity
provider's address, and upper and lower bounds of the position.
1. Each pool contract will also need to maintain a ticks registry and information about each tick–the amount of liquidity
provided by each tick.
1. Since the tick range is limited, we need to store the limits in the contract, as constants.
1. And as we discussed in the introduction, pool contracts store the amount of liquidity, $L$, and $\sqrt{P}$ instead of
token reserves. So we'll need to store them in the contract as well.

Here's what our pool contract with all the state variables:

```solidity
contract UniswapV3Pool {
    using Tick for mapping(int24 => Tick.Info);
    using Position for mapping(bytes32 => Position.Info);
    using Position for Position.Info;

    int24 internal constant MIN_TICK = -887272;
    int24 internal constant MAX_TICK = -MIN_TICK;

    // Pool tokens, immutable
    address public immutable token0;
    address public immutable token1;

    // First slot will contain essential data
    struct Slot0 {
        // Current sqrt(P)
        uint160 sqrtPriceX96;
        // Current tick
        int24 tick;
    }
    Slot0 public slot0;

    // Amount of liquidity, L.
    uint128 public liquidity;

    // Ticks info
    mapping(int24 => Tick.Info) public ticks;
    // Positions info
    mapping(bytes32 => Position.Info) public positions;

    ...
```

And we'll initialize them in the constructor:

```solidity
    constructor(
        address token0_,
        address token1_,
        uint160 sqrtPriceX96,
        int24 tick
    ) {
        token0 = token0_;
        token1 = token1_;

        slot0 = Slot0({sqrtPriceX96: sqrtPriceX96, tick: tick});
    }
}
```

`Tick` and `Position` are custom types that make the managing of ticks and positions easier. You'll find their code in `lib`
folder.

> For brevity, I'll omit detailed explanation of Solidity syntax and features. Solidity has [great documentation](https://docs.soliditylang.org/en/latest/),
don't hesitate referring to it if something is not clear!

This is our starting point, and our goal in this chapter is to make our first swap.

## Liquidity Calculation
Trading is not possible without liquidity, and to make our first swap we need to put some liquidity into the pool contract.
Providing liquidity into a pool simply means:
- sending tokens to the pool,
- updating pool parameters so our liquidity is recorded and used for swaps.

Tokens are smart contracts, they track balances inside their own storage: when you send tokens to someone you simply
update balances in token's storage. Thus, we don't need to enable tokens receiving in our pool contract–it'll just
work! However, we want the contract to put provided liquidity into some price range.

Remember that, in Uniswap V3, the entire price range is demaracted into *ticks*: each tick corresponds to a price and has
an index. For the sake of simplicity, let's pretend that we're going to buy ETH for USDC at the price of <span>$5000</span>
per 1 ETH. And we're going to buy some amount of ETH to push the price slightly above <span>$5000</span>. We want
to provide liquidity at a range that includes this price. And we want to be sure that final price will stay **within this
range** (again, let's keep it as simple as possible at this moment).

Let's find the index of the tick that corresponds to the price of $5000 per 1 ETH. We know that:

$$\sqrt{p(i)}=1.0001^{\frac{i}{2}}$$

Thus, to find $i$:

$$i = log_{\sqrt{1.0001}} \sqrt{p(i)}$$

Since we're going to buy ETH, we're going to sell USDC. Thus the price is:

$$p = \frac{y}{x} = \frac{1}{5000} = 0.0002 \enspace ETH/USDC$$

We can find the tick:

$$i = log_{\sqrt{1.0001}}\sqrt{0.0002} = log_{1.00005}0.014142135 = -85174.062077$$

And we'll then round it down to nearest integer:

$$i = -85174$$

This tick will be the lower bound of the price range we'll provide liquidity into. How to find the upper bound? We can
choose whichever we want! Let's choose the span of 100 ticks:

$$[-85174;-85074]$$

[TODO: 100 or 101?]

Let's see how we can convert the upper tick to the corresponding price:
1. First, we know that the difference in $\sqrt{p}$ of two adjacent ticks is 1 basis point (0.01% or 0.0001).
1. Since our range is 100 ticks, the price of $\sqrt{p(i+100)}$ is:
$$\sqrt{p(i+100)} = \sqrt{p(i)}*1.0001^{100}$$
1. Let's find it:
$$\sqrt{p(-85074)} = \sqrt{p(-85174)}*1.0001^{100} = \sqrt{0.0002}*1.0001^{100} \approx 0,0142842$$
$$p(-85074) \approx 0,0142842^{2} \approx 0,00020404 \enspace ETH/USDC$$

Expectedly, the price is higher! And the reciprocal price is expectedly lower:
$$\frac{1}{0.00020404} = 4901 \enspace USDC/ETH$$

So, if we pick this price range, the curve within this range will look like this:

[TODO: add curve]

> At this point, we'll only implement a swapping within this curve and price range. Later on, we'll see how swapping works
when current price range doesn't have enough liquidity.

Lastly, we need to calculate the amount of liquidity that will be enough to make a small trade within the price range. We
expect to start at 0.0002 ETH/USDC and bring the price not higher than 0.00020404 ETH/USDC:

[TODO: add curve with start and end prices]

To not overload this chapter with calculations, we'll simply pick it up empirically 🤷‍♂️ During development and testing,
we'll be able to mint as many tokens as we want.

## Minting

The process of providing liquidity in Uniswap V2 is called *minting*. The reason is that the V2 pool contract mints
tokens (LP-tokens) in exchange for liquidity. V3 doesn't do that, but it still uses the same name for the function. Let's
use it as well:

```solidity
function mint(
    address owner,
    int24 lowerTick,
    int24 upperTick,
    uint128 amount
) external returns (uint256 amount0, uint256 amount1) {
    ...
```

Our `mint` function will take:
1. Owner's address, to track the owner of the liquidity.
1. Upper and lower ticks, to set the bounds of a price range.
1. The amount of liquidity we have provided.

When adding initial liquidity to a pool, this function adds a new tick and a position.

We begin with checking the ticks:
```solidity
if (
    lowerTick >= upperTick ||
    lowerTick < MIN_TICK ||
    upperTick > MAX_TICK
) revert InvalidTickRange();
```

And ensuring that some amount of liquidity is provided:
```solidity
if (amount == 0) revert ZeroLiquidity();
```

Then, add a tick and a position:
```solidity
ticks.update(lowerTick, amount);
ticks.update(upperTick, amount);

Position.Info storage position = positions.get(
    owner,
    lowerTick,
    upperTick
);
position.update(amount);
```

The `ticks.update` function is:
```solidity
// src/libs/Tick.sol
...
function update(
    mapping(int24 => Tick.Info) storage self,
    int24 tick,
    int128 liquidityDelta
) internal {
    Tick.Info storage tickInfo = self[tick];
    uint128 liquidityBefore = tickInfo.liquidity;
    uint128 liquidityAfter = liquidityBefore + uint128(liquidityDelta);

    if (liquidityBefore == 0) {
        tickInfo.initialized = true;
    }

    tickInfo.liquidity = liquidityAfter;
}
...
```
It initialized a tick if it has 0 liquidity before and adds new liquidity to it. As you can see, we're calling this
function on both lower and upper ticks, thus liquidity is added to both of them–we'll see why later on.

The `position.update` function is:
```solidity
// src/libs/Position.sol
function update(Info storage self, int128 liquidityDelta) internal {
    uint128 liquidityBefore = self.liquidity;
    uint128 liquidityAfter = liquidityBefore + uint128(liquidityDelta);

    self.liquidity = liquidityAfter;
}
```
Similar to the tick update function, it adds liquidity to a specific position. And to get a position we call:
```solidity
// src/libs/Position.sol
...
function get(
    mapping(bytes32 => Info) storage self,
    address owner,
    int24 lowerTick,
    int24 upperTick
) internal view returns (Position.Info storage position) {
    position = self[
        keccak256(abi.encodePacked(owner, lowerTick, upperTick))
    ];
}
...
```

Each position is uniquely identified by three keys: owner address, lower tick index, and upper tick index. We're storing
positions in a `bytes32 => Info` map and are using hashes of concatenated owner address, lower tick, and upper tick as
keys. This is cheaper than storing three nested maps.

We're not done yet! Next, we need to calculate the amounts that a user must deposit. These amounts will be based on current
$\sqrt{P}$ and the parameters the user passed to `mint`.
```solidity
```