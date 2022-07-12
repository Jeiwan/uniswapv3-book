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

{{< katex display >}} {{</ katex >}}

# Math in Solidity

Due to Solidity not supporting float-point and fixed-point numbers, math in Solidity is somewhat complicated. Solidity
gives us integer and unsigned integer types, which are not enough for for more or less complex math calculations.

Another difficulty is gas consumption: the more complex an algorithm, the more gas it consumes. Thus, if we need to have
advanced math operations (like `exp`, `ln`, `sqrt`), we want them to be as gas efficient as possible.

And another big problem is the possibility of under/overflow. When multiplying `uint256` numbers, there's a risk of an
overflow: the result number might be so big that it won't fit into 256 bits.

All these difficulties force us to use third-party math libraries that implement advanced math operations and, ideally,
optimize their gas consumption. In the case when there's no library for an algorithm we need, we'll have to implement
it ourselves, and will need to know all the ins and outs of the math algorithm.

## Re-using Math Contracts

In our Uniswap V3 implementation, we're going to use two third-party math contracts:
1. [PRBMath](https://github.com/paulrberg/prb-math), which is a great library of advanced fixed-point math algorithms.
We'll use `mulDiv` function to handle overflows when multiplying and then dividing integer numbers.
1. [TickMath](https://github.com/Uniswap/v3-core/blob/main/contracts/libraries/TickMath.sol) from the original Uniswap
V3 repo. This contract implements two functions, `getSqrtRatioAtTick` and `getTickAtSqrtRatio`, which convert between
ticks and corresponding $\sqrt{P}$.

Let's focus on the latter.

In our contracts, we'll need to convert ticks to corresponding $\sqrt{P}$ and back. The formulas of finding $\sqrt{P}$
at tick $i$ and vice versa:

$$\sqrt{P(i)} = \sqrt{1.0001^i} = 1.0001^{\frac{i}{2}}$$

$$i = log_{\sqrt{1.0001}}\sqrt{P(i)}$$

The first formula takes square root of a number with a fractional part. The second one takes a logarithm of a similar
number. These are advanced mathematical operations that are not implemented in Solidity. Moreover, these formulas are
specific to Uniswap V3: they're both based on taking square roots of powers of 1.0001. This means that we can implement
a specific function finds only those that are based on powers of 1.0001 and not all logarithms and square root. Moreover,
we know that tick indices are limited ($[−887272,887272]$), and this means our implementation doesn't need to calculate
them outside of the range.

This leads us to a conclusion: we need to implement our own mathematical function. Well, two of them.

If you look at the original code of [getSqrtRatioAtTick](https://github.com/Uniswap/v3-core/blob/8f3e4645a08850d2335ead3d1a8d0c64fa44f222/contracts/libraries/TickMath.sol#L23-L54)
and [getTickAtSqrtRatio](https://github.com/Uniswap/v3-core/blob/8f3e4645a08850d2335ead3d1a8d0c64fa44f222/contracts/libraries/TickMath.sol#L61-L204)
you'll see that they're quite complex: there're a lot of magic numbers (like `0xfffcb933bd6fad37aa2d162d1a594001`),
multiplication, and bitwise operations. At this point, we're not going to analyze the code or re-implement it since this
is a very advanced and somewhat different topic. We'll use the contract as is.

The only thing you need to know about these functions is that they're **approximations** of the above formulas on the
tick indices range. One of them approximates $\sqrt{P(i)} = \sqrt{1.0001^i}$ and the other approximates
$i = log_{\sqrt{1.0001}}\sqrt{P(i)}$.