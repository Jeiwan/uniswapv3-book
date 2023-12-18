# Generalized Minting

Now, we're ready to update `mint` function so we don't need to hard code values anymore and can calculate them instead.


## Indexing Initialized Ticks

Recall that, in the `mint` function, we update the TickInfo mapping to store information about available liquidity at ticks.  Now, we also need to index newly initialized ticks in the bitmap index–we'll later use this index to find next initialized tick during swapping.

First, we need to update `Tick.update` function:
```solidity
// src/lib/Tick.sol
function update(
    mapping(int24 => Tick.Info) storage self,
    int24 tick,
    uint128 liquidityDelta
) internal returns (bool flipped) {
    ...
    flipped = (liquidityAfter == 0) != (liquidityBefore == 0);
    ...
}
```

It now returns `flipped` flag, which is set to true when liquidity is added to an empty tick or when entire liquidity is removed from a tick.

Then, in `mint` function, we update the bitmap index:
```solidity
// src/UniswapV3Pool.sol
...
bool flippedLower = ticks.update(lowerTick, amount);
bool flippedUpper = ticks.update(upperTick, amount);

if (flippedLower) {
    tickBitmap.flipTick(lowerTick, 1);
}

if (flippedUpper) {
    tickBitmap.flipTick(upperTick, 1);
}
...
```

> Again, we're setting tick spacing to 1 until we introduce different values in Milestone 4.

## Token Amounts Calculation

The biggest change in `mint` function is switching to tokens amount calculation. In Milestone 1, we hard coded these values:
```solidity
    amount0 = 0.998976618347425280 ether;
    amount1 = 5000 ether;
```

And now we're going to calculate them in Solidity using formulas from Milestone 1. Let's recall those formulas:

$$\Delta x = \frac{L(\sqrt{p(i_u)} - \sqrt{p(i_c)})}{\sqrt{p(i_u)}\sqrt{p(i_c)}}$$
$$\Delta y = L(\sqrt{p(i_c)} - \sqrt{p(i_l)})$$

$\Delta x$ is the amount of `token0`, or token $x$. Let's implement it in Solidity:
```solidity
// src/lib/Math.sol
function calcAmount0Delta(
    uint160 sqrtPriceAX96,
    uint160 sqrtPriceBX96,
    uint128 liquidity
) internal pure returns (uint256 amount0) {
    if (sqrtPriceAX96 > sqrtPriceBX96)
        (sqrtPriceAX96, sqrtPriceBX96) = (sqrtPriceBX96, sqrtPriceAX96);

    require(sqrtPriceAX96 > 0);

    amount0 = divRoundingUp(
        mulDivRoundingUp(
            (uint256(liquidity) << FixedPoint96.RESOLUTION),
            (sqrtPriceBX96 - sqrtPriceAX96),
            sqrtPriceBX96
        ),
        sqrtPriceAX96
    );
}
```

> This function is identical to `calc_amount0` in our Python script.

First step is to sort the prices to ensure we don't underflow when subtracting. Next, we convert `liquidity` to a Q96.64 number by multiplying it by 2**96. Next, according to the formula, we multiply it by the difference of the prices and divide it by the bigger price. Then, we divide by the smaller price. The order of division doesn't matter, but we want to have two divisions because multiplication of prices can overflow.

We're using `mulDivRoundingUp` to multiply and divide in one operation. This function is based on `mulDiv` from `PRBMath`:
```solidity
function mulDivRoundingUp(
    uint256 a,
    uint256 b,
    uint256 denominator
) internal pure returns (uint256 result) {
    result = PRBMath.mulDiv(a, b, denominator);
    if (mulmod(a, b, denominator) > 0) {
        require(result < type(uint256).max);
        result++;
    }
}
```

`mulmod` is a Solidity function that multiplies two numbers (`a` and `b`), divides the result by `denominator`, and returns the remainder. If the remainder is positive, we round the result up.

Next, $\Delta y$:
```solidity
function calcAmount1Delta(
    uint160 sqrtPriceAX96,
    uint160 sqrtPriceBX96,
    uint128 liquidity
) internal pure returns (uint256 amount1) {
    if (sqrtPriceAX96 > sqrtPriceBX96)
        (sqrtPriceAX96, sqrtPriceBX96) = (sqrtPriceBX96, sqrtPriceAX96);

    amount1 = mulDivRoundingUp(
        liquidity,
        (sqrtPriceBX96 - sqrtPriceAX96),
        FixedPoint96.Q96
    );
}
```

> This function is identical to `calc_amount1` in our Python script.

Again, we're using `mulDivRoundingUp` to avoid overflows during multiplication.

And that's it! We can now use the functions to calculate token amounts:
```solidity
// src/UniswapV3Pool.sol
function mint(...) {
    ...
    Slot0 memory slot0_ = slot0;

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
    ...
}
```

Everything else remains the same. You'll need to update the amounts in the pool tests, they'll be slightly different due to rounding.