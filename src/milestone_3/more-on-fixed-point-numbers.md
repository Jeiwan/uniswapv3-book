# A Little Bit More on Fixed-point Numbers

In this bonus chapter, I'd like to show you how to convert prices to ticks in Solidity. We don't need to do this in the main contracts, but it's helpful to have such function in tests so we don't hardcode ticks and could write something like `tick(5000)`–this makes code easier to read because it's more convenient for us to think in prices, not tick indexes.

Recall that, to find ticks, we use `TickMath.getTickAtSqrtRatio` function, which takes $\sqrt{P}$ as its argument, and the $\sqrt{P}$ is a Q64.96 fixed-point number. In smart contract tests, we need to check $\sqrt{P}$ many times in many different test cases: mostly after mints and swaps. Instead of hard coding actual values, it might be cleaner to use a helper function like `sqrtP(5000)` that converts prices to $\sqrt{P}$.

So, what's the problem?

The problem is that Solidity doesn't natively support the square root operation, which means we need a third-party library. Another problem is that prices are often relatively small numbers, like 10, 5000, 0.01, etc., and we don't want to lose precision when taking square root.

You probably remember that we used `PRBMath` earlier in the book to implement multiply-then-divide operation that doesn't overflow during multiplication. If you check `PRBMath.sol` contract, you'll notice `sqrt` function. However, the function doesn't support fixed-point numbers, as the function description says. You can give it a try and see that `PRBMath.sqrt(5000)` results in `70`, which is an integer number with lost precision (without the fractional part).

If you check [prb-math](https://github.com/paulrberg/prb-math) repo, you'll see these contracts: `PRBMathSD59x18.sol` and `PRBMathUD60x18.sol`. Aha! These are fixed-point number implementations. Let's pick the latter and see how it goes: `PRBMathUD60x18.sqrt(5000 * PRBMathUD60x18.SCALE)` returns `70710678118654752440`. This looks interesting!  `PRBMathUD60x18` is a library that implements fixed-numbers with 18 decimal places in the fractional part. So the number we got is actually 70.710678118654752440 (use `cast --from-wei 70710678118654752440`).

However, we cannot use this number!

There are fixed-point numbers and fixed-point numbers. The Q64.96 fixed-point number used by Uniswap V3 is a **binary** number–64 and 96 signify *binary places*. But `PRBMathUD60x18` implements a *decimal* fixed-point number (UD in the contract name means "unsigned, decimal"), where 60 and 18 signify *decimal places*. This difference is quite significant.

Let's see how to convert an arbitrary number (42) to either of the above fixed-point numbers:
1. Q64.96: $42 * 2^{96}$ or, using bitwise left shift, `2 << 96`. The result is 3327582825599102178928845914112.
1. UD60.18: $42 * 10^{18}$. The result is 42000000000000000000.

Let's now see how to convert numbers with the fractional part (42.1337):
1. Q64.96: $421337 * 2^{92}$ or `421337 << 92`. The result is 2086359769329537075540689212669952.
1. UD60.18: $421337 * 10^{14}$. The result is 42133700000000000000.

The second variant makes more sense to us because it uses the decimal system, which we learned in our childhood. The first variant uses the binary system and it's much harder for us to read.

But the biggest problem with different variants is that it's hard to convert between them.

This all means that we need a different library, one that implements a binary fixed-point number and `sqrt` function for it. Luckily, there's such library: [abdk-libraries-solidity](https://github.com/abdk-consulting/abdk-libraries-solidity).  The library implemented Q64.64, not exactly what we need (not 96 bits in the fractional part) but this is not a problem.

Here's how we can implement the price-to-tick function using the new library:
```solidity
function tick(uint256 price) internal pure returns (int24 tick_) {
    tick_ = TickMath.getTickAtSqrtRatio(
        uint160(
            int160(
                ABDKMath64x64.sqrt(int128(int256(price << 64))) <<
                    (FixedPoint96.RESOLUTION - 64)
            )
        )
    );
}
```

`ABDKMath64x64.sqrt` takes Q64.64 numbers so we need to convert `price` to such number. The price is expected to not have the fractional part, so we're shifting it by 64 bits. The `sqrt` function also returns a Q64.64 number but `TickMath.getTickAtSqrtRatio` takes a Q64.96 number–this is why we need to shift the result of the square root operation by `96 - 64` bits to the left.