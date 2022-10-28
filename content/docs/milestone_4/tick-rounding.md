---
title: "Tick Rounding"
weight: 6
# bookFlatSection: false
# bookToc: true
# bookHidden: false
# bookCollapseSection: false
# bookComments: false
# bookSearchExclude: false
---

# Tick Rounding

Let's review some other changes we need to make to support different tick spacings.

Tick spacing greater than 1 won't allow users to select arbitrary price ranges: tick indexes must be multiples of a tick
spacing. For example, for tick spacing 60 we can have ticks: 0, 60, 120, 180, etc. Thus, when user picks a range, we need
to "round" it so its boundaries are multiples of pool's tick spacing.

## `nearestUsableTick` in JavaScript
In [the Uniswap V3 SDK](https://github.com/Uniswap/v3-sdk), the function that does that is called [nearestUsableTick](https://github.com/Uniswap/v3-sdk/blob/b6cd73a71f8f8ec6c40c130564d3aff12c38e693/src/utils/nearestUsableTick.ts):
```javascript
/**
 * Returns the closest tick that is nearest a given tick and usable for the given tick spacing
 * @param tick the target tick
 * @param tickSpacing the spacing of the pool
 */
export function nearestUsableTick(tick: number, tickSpacing: number) {
  invariant(Number.isInteger(tick) && Number.isInteger(tickSpacing), 'INTEGERS')
  invariant(tickSpacing > 0, 'TICK_SPACING')
  invariant(tick >= TickMath.MIN_TICK && tick <= TickMath.MAX_TICK, 'TICK_BOUND')
  const rounded = Math.round(tick / tickSpacing) * tickSpacing
  if (rounded < TickMath.MIN_TICK) return rounded + tickSpacing
  else if (rounded > TickMath.MAX_TICK) return rounded - tickSpacing
  else return rounded
}
```

At its core, it's just:
```javascript
Math.round(tick / tickSpacing) * tickSpacing
```

Where `Math.round` is rounding to the nearest integer: when the fractional part is less than 0.5, it rounds to the lower
integer; when it's greater than 0.5 it rounds to the greater integer; and when it's 0.5, it rounds to the greater integer
as well.

So, in the web app, we'll use `nearestUsableTick` when building `mint` parameters:
```javascript
const mintParams = {
  tokenA: pair.token0.address,
  tokenB: pair.token1.address,
  tickSpacing: pair.tickSpacing,
  lowerTick: nearestUsableTick(lowerTick, pair.tickSpacing),
  upperTick: nearestUsableTick(upperTick, pair.tickSpacing),
  amount0Desired, amount1Desired, amount0Min, amount1Min
}
```

> In reality, it should be called whenever user adjusts a price range because we want the user to see the actual price
that will be created. In our simplified app, we do it less user-friendly.

However, we also want to have a similar function in Solidity tests, but neither of the math libraries we're using
implements it.

## `nearestUsableTick` in Solidity

In our smart contract tests, we need a way to round ticks and convert rounded prices to $\sqrt{P}$. In a previous chapter, we
chose to use [ABDKMath64x64](https://github.com/abdk-consulting/abdk-libraries-solidity) to handle fixed-point numbers
math in tests. The library, however, doesn't implement the rounding function we need to port `nearestUsableTick`, so
we'll need to implement it ourselves:

```solidity
function divRound(int128 x, int128 y)
    internal
    pure
    returns (int128 result)
{
    int128 quot = ABDKMath64x64.div(x, y);
    result = quot >> 64;

    // Check if remainder is greater than 0.5
    if (quot % 2**64 >= 0x8000000000000000) {
        result += 1;
    }
}
```

The function does multiple things:
1. it divides two Q64.64 numbers;
1. it then rounds the result to the decimal one (`result = quot >> 64`), the fractional part is lost at this point (i.e. the result is rounded down);
1. it then divides the quotient by $2^{64}$, takes the remainder, and compares it with `0x8000000000000000` (which is 0.5 in Q64.64);
1. if the remainder is greater or equal to 0.5, it rounds the result to the greater integer.

What we get is an integer rounded according to the rules of `Math.round` from JavaScript. We can then re-implement
`nearestUsableTick`:

```solidity
function nearestUsableTick(int24 tick_, uint24 tickSpacing)
    internal
    pure
    returns (int24 result)
{
    result =
        int24(divRound(int128(tick_), int128(int24(tickSpacing)))) *
        int24(tickSpacing);

    if (result < TickMath.MIN_TICK) {
        result += int24(tickSpacing);
    } else if (result > TickMath.MAX_TICK) {
        result -= int24(tickSpacing);
    }
}
```

That's it!

{{< katex display >}} {{</ katex >}}

