---
title: "Math in Solidity"
weight: 3
# bookFlatSection: false
# bookToc: true
# bookHidden: false
# bookCollapseSection: false
# bookComments: false
# bookSearchExclude: false
---

# Math in Solidity

Due to Solidity not supporting numbers with th fractional part, math in Solidity is somewhat complicated. Solidity
gives us integer and unsigned integer types, which are not enough for for more or less complex math calculations.

Another difficulty is gas consumption: the more complex an algorithm, the more gas it consumes. Thus, if we need to have
advanced math operations (like `exp`, `ln`, `sqrt`), we want them to be as gas efficient as possible.

And another big problem is the possibility of under/overflow. When multiplying `uint256` numbers, there's a risk of an
overflow: the result number might be so big that it won't fit into 256 bits.

All these difficulties force us to use third-party math libraries that implement advanced math operations and, ideally,
optimize their gas consumption. In the case when there's no library for an algorithm we need, we'll have to implement
it ourselves, which is a difficult task if we need to implement a unique computation.

## Re-using Math Contracts

In our Uniswap V3 implementation, we're going to use two third-party math contracts:
1. [PRBMath](https://github.com/paulrberg/prb-math), which is a great library of advanced fixed-point math algorithms.
We'll use `mulDiv` function to handle overflows when multiplying and then dividing integer numbers.
1. [TickMath](https://github.com/Uniswap/v3-core/blob/main/contracts/libraries/TickMath.sol) from the original Uniswap
V3 repo. This contract implements two functions, `getSqrtRatioAtTick` and `getTickAtSqrtRatio`, which convert $\sqrt{P}$'s
to ticks and back.

Let's focus on the latter.

In our contracts, we'll need to convert ticks to corresponding $\sqrt{P}$ and back. The formulas are:

$$\sqrt{P(i)} = \sqrt{1.0001^i} = 1.0001^{\frac{i}{2}}$$

$$i = log_{\sqrt{1.0001}}\sqrt{P(i)}$$

These are complex mathematical operations (for Solidity, at least) and they require high precision because we don't want
to allow rounding errors when calculating prices. To have better precision and optimization we'll need unique implementation.

If you look at the original code of [getSqrtRatioAtTick](https://github.com/Uniswap/v3-core/blob/8f3e4645a08850d2335ead3d1a8d0c64fa44f222/contracts/libraries/TickMath.sol#L23-L54)
and [getTickAtSqrtRatio](https://github.com/Uniswap/v3-core/blob/8f3e4645a08850d2335ead3d1a8d0c64fa44f222/contracts/libraries/TickMath.sol#L61-L204)
you'll see that they're quite complex: there're a lot of magic numbers (like `0xfffcb933bd6fad37aa2d162d1a594001`),
multiplication, and bitwise operations. At this point, we're not going to analyze the code or re-implement it since this
is a very advanced and somewhat different topic. We'll use the contract as is. And, in a later milestone, we'll break
down the computations.

{{< katex display >}} {{</ katex >}}