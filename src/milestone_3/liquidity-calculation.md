# Liquidity Calculation

Of the whole math of Uniswap V3, what we haven't yet implemented in Solidity is liquidity calculation. In the Python script, we have these functions:

```python
def liquidity0(amount, pa, pb):
    if pa > pb:
        pa, pb = pb, pa
    return (amount * (pa * pb) / q96) / (pb - pa)


def liquidity1(amount, pa, pb):
    if pa > pb:
        pa, pb = pb, pa
    return amount * q96 / (pb - pa)
```

Let's implement them in Solidity so we could calculate liquidity in the `Manager.mint()` function.

## Implementing Liquidity Calculation for Token X

The functions we're going to implement allow us to calculate liquidity ($L = \sqrt{xy}$) when token amounts and price ranges are known. Luckily, we already know all the formulas. Let's recall this one:

$$\Delta x = \Delta \frac{1}{\sqrt{P}}L$$

In a previous chapter, we used this formula to calculate swap amounts ($\Delta x$ in this case) and now we're going to use it to find $L$:

$$L = \frac{\Delta x}{\Delta \frac{1}{\sqrt{P}}}$$

Or, after simplifying it:
$$L = \frac{\Delta x \sqrt{P_u} \sqrt{P_l}}{\sqrt{P_u} - \sqrt{P_l}}$$

> We derived this formula in [Liquidity Amount Calculation](https://uniswapv3book.com/docs/milestone_1/calculating-liquidity/#liquidity-amount-calculation).

In Solidity, we'll again use `PRBMath` to handle overflows when multiplying and then dividing:

```solidity
function getLiquidityForAmount0(
    uint160 sqrtPriceAX96,
    uint160 sqrtPriceBX96,
    uint256 amount0
) internal pure returns (uint128 liquidity) {
    if (sqrtPriceAX96 > sqrtPriceBX96)
        (sqrtPriceAX96, sqrtPriceBX96) = (sqrtPriceBX96, sqrtPriceAX96);

    uint256 intermediate = PRBMath.mulDiv(
        sqrtPriceAX96,
        sqrtPriceBX96,
        FixedPoint96.Q96
    );
    liquidity = uint128(
        PRBMath.mulDiv(amount0, intermediate, sqrtPriceBX96 - sqrtPriceAX96)
    );
}
```

## Implementing Liquidity Calculation for Token Y

Similarly, we'll use the other formula from [Liquidity Amount Calculation](https://uniswapv3book.com/docs/milestone_1/calculating-liquidity/#liquidity-amount-calculation) to find $L$ when amount of $y$ and price range is known:

$$\Delta y = \Delta\sqrt{P} L$$
$$L = \frac{\Delta y}{\sqrt{P_u}-\sqrt{P_l}}$$

```solidity
function getLiquidityForAmount1(
    uint160 sqrtPriceAX96,
    uint160 sqrtPriceBX96,
    uint256 amount1
) internal pure returns (uint128 liquidity) {
    if (sqrtPriceAX96 > sqrtPriceBX96)
        (sqrtPriceAX96, sqrtPriceBX96) = (sqrtPriceBX96, sqrtPriceAX96);

    liquidity = uint128(
        PRBMath.mulDiv(
            amount1,
            FixedPoint96.Q96,
            sqrtPriceBX96 - sqrtPriceAX96
        )
    );
}
```

I hope this is clear!

## Finding Fair Liquidity

You might be wondering why there are two ways of calculating $L$ while we have always had only one $L$, which is calculated as $L = \sqrt{xy}$, and which of these ways is correct. The answer is: they're both correct.

In the above formulas, we calculate $L$ based on different parameters: price range and the amount of either token.  Different price ranges and different token amounts will result in different values of $L$. And there's a scenario where we need to calculate both of the $L$'s and pick one of them. Recall this piece from `mint` function:

```solidity
if (slot0_.tick < lowerTick) {
    amount0 = Math.calcAmount0Delta(...);
} else if (slot0_.tick < upperTick) {
    amount0 = Math.calcAmount0Delta(...);

    amount1 = Math.calcAmount1Delta(...);

    liquidity = LiquidityMath.addLiquidity(liquidity, int128(amount));
} else {
    amount1 = Math.calcAmount1Delta(...);
}
```

It turns out, we also need to follow this logic when calculating liquidity:
1. if we're calculating liquidity for a range that's above the current price, we use the $\Delta x$ version on the formula;
1. when calculation liquidity for a range that's below the current price, we use the $\Delta y$ one;
1. when a price range includes the current price, we calculate **both** and pick the smaller of them.

> Again, we discussed these ideas in [Liquidity Amount Calculation](https://uniswapv3book.com/docs/milestone_1/calculating-liquidity/#liquidity-amount-calculation).

Let's implement this logic now.

When current price is below the lower bound of a price range:
```solidity
function getLiquidityForAmounts(
    uint160 sqrtPriceX96,
    uint160 sqrtPriceAX96,
    uint160 sqrtPriceBX96,
    uint256 amount0,
    uint256 amount1
) internal pure returns (uint128 liquidity) {
    if (sqrtPriceAX96 > sqrtPriceBX96)
        (sqrtPriceAX96, sqrtPriceBX96) = (sqrtPriceBX96, sqrtPriceAX96);

    if (sqrtPriceX96 <= sqrtPriceAX96) {
        liquidity = getLiquidityForAmount0(
            sqrtPriceAX96,
            sqrtPriceBX96,
            amount0
        );
```

When current price is within a range, we're picking the smaller $L$:
```solidity
} else if (sqrtPriceX96 <= sqrtPriceBX96) {
    uint128 liquidity0 = getLiquidityForAmount0(
        sqrtPriceX96,
        sqrtPriceBX96,
        amount0
    );
    uint128 liquidity1 = getLiquidityForAmount1(
        sqrtPriceAX96,
        sqrtPriceX96,
        amount1
    );

    liquidity = liquidity0 < liquidity1 ? liquidity0 : liquidity1;
```

And finally:
```solidity
} else {
    liquidity = getLiquidityForAmount1(
        sqrtPriceAX96,
        sqrtPriceBX96,
        amount1
    );
}
```

Done.