---
title: "Different Price Ranges"
weight: 2
# bookFlatSection: false
# bookToc: true
# bookHidden: false
# bookCollapseSection: false
# bookComments: false
# bookSearchExclude: false
---

{{< katex display >}} {{</ katex >}}

# Different Price Ranges

The way we implemented it, our Pool contract creates only price ranges that include the current price:
```solidity
// src/UniswapV3Pool.sol
function mint(
    ...
    amount0 = Math.calcAmount0Delta(
        slot0_.sqrtPriceX96,
        TickMath.getSqrtRatioAtTick(upperTick),
        amount
    );

    amount1 = Math.calcAmount1Delta(
        slot0_.sqrtPriceX96,
        TickMath.getSqrtRatioAtTick(lowerTick),
        amount
    );

    liquidity += uint128(amount);
    ...
}
```

From this piece you can also see that we always update the liquidity tracker (which tracks only currently available
liquidity, i.e. liquidity available at the current price).

However, in reality, price ranges can also be created **below or above** the current price. That's it: the design of
Uniswap V3 allows liquidity provider to provide liquidity that doesn't get immediately used; such liquidity gets "injected"
when current price gets into the "sleeping" price ranges.

These are kinds of price ranges that can exist:
1. Active price range, i.e. one that includes current price.
1. Price range placed below current price. The upper tick of this range is below the current tick.
1. Price range placed above current price. The lower tick of this range is above the current tick.

## Limit Orders

An interesting fact about inactive liquidity (i.e. liquidity not provided at current price) is that it acts as *limit
orders*.

In trading, limit orders are orders that get executed when price crosses a level chosen by trader. For example, you can
place a limit order that buys 1 ETH when its price drops to $1000. Similarly, you can use limit order to sell assets.
With Uniswap V3, you can get similar behavior by placing liquidity at ranges that are below or above current price. Let's
see how this works:

[TODO: illustrate liquidity to the left and to the right of the current price]

If you provide liquidity below current price (i.e. the price range you chose lays entirely below the current price) or
above it, then your whole liquidity will be composed of **only one asset**–the asset will be the cheapest one of the two.
In our example, we're building a pool with ETH being token $X$ and USDC being token $Y$, and we define the price as:

$$P = \frac{y}{x}$$

If we put liquidity below current price, then the liquidity will be composed of USDC solely because, where we added the
liquidity, the price of USDC is lower than the current price. Likewise, when we put liquidity above current price, then
the liquidity will be composed of ETH because ETH is cheaper in that range.

If this sounds confusing, let's look at one price range in isolation:

[TODO: illustrate a price range, curve with current/lower/upper prices]

If we buy all available amount of token $X$ from this range, the range will contain only the other token, token $Y$, and
the price will move to the left of the curve. The price, as we defined it, will **increase**. If there's a price range
to the left of this one, it needs to have $X$ liquidity, and only $X$, not $Y$: it needs to provide $Y$ for our swap.
If we keep buying and rising the price, we might "drain" this price as well, which means buying all its $X$ tokens and
selling $Y$ tokens. Again, the price range ends up having only $Y$ token and current price moves outside of it.

Similarly, if we're buying $Y$ token, we move the price to the right and removing $Y$ tokens from the pool. The next
price range will only contain $Y$ tokens to satisfy our demand, and, similarly to the above scenario, will contain only
$X$ tokens if we buy all $Y$ tokens from it.

Note the interesting fact: when crossing an entire price range, it's liquidity is swapped from one token to another. And
if we set a very narrow price range, one that gets crossed quickly during a price move, we get a limit order! For example,
if you want to buy ETH at a lower price, you need to place a price range containing only USDC at the lower price and
wait for current price to cross it. After that, you'll need to remove your liquidity and get it whole converted to ETH!

I hope this didn't confuse you! Let's get to code.

## Updating `mint` Function

To support all the kings of price ranges, we need to current price is below, inside, or above the price range specified
by user and calculate token amounts accordingly. If the price range is above the current price, we want the liquidity
to be composed of token $X$:

```solidity
// src/UniswapV3Pool.sol
function mint(
    ...
    if (slot0_.tick < lowerTick) {
        amount0 = Math.calcAmount0Delta(
            TickMath.getSqrtRatioAtTick(lowerTick),
            TickMath.getSqrtRatioAtTick(upperTick),
            amount
        );
    ...
```

When the price range includes the current price, we want both tokens in amounts proportional to the price (this is the
scenario we implemented earlier):
```solidity
} else if (slot0_.tick < upperTick) {
    amount0 = Math.calcAmount0Delta(
        slot0_.sqrtPriceX96,
        TickMath.getSqrtRatioAtTick(upperTick),
        amount
    );

    amount1 = Math.calcAmount1Delta(
        slot0_.sqrtPriceX96,
        TickMath.getSqrtRatioAtTick(lowerTick),
        amount
    );

    liquidity = LiquidityMath.addLiquidity(liquidity, int128(amount));
```
Notice that this is the only scenario where we want to update `liquidity` since the variable tracks liquidity that's
available immediately.

In all other cases, when the price range is below the current price, we want the range to contain only token $Y$:
```solidity
} else {
    amount1 = Math.calcAmount1Delta(
        TickMath.getSqrtRatioAtTick(lowerTick),
        TickMath.getSqrtRatioAtTick(upperTick),
        amount
    );
}
```

And that's it!