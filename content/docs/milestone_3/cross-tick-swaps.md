---
title: "Cross-Tick Swaps"
weight: 3
# bookFlatSection: false
# bookToc: true
# bookHidden: false
# bookCollapseSection: false
# bookComments: false
# bookSearchExclude: false
---

{{< katex display >}} {{</ katex >}}

# Cross-Tick Swaps

Cross-tick swaps is probably the most advanced feature of Uniswap V3. Luckily, we have already implemented almost everything
we need to make cross-tick swaps. Let's review how swapping works in our current implementation.

## How Cross-Tick Swaps Work

Let's recap how cross-tick swaps work before getting down to code.

[TODO: illustrate a pool with multiple price ranges]

A common Uniswap V3 pool is a pool with many overlapping (and outstanding) price ranges. Each pool tracks current $\sqrt{P}$
and tick. When users swap tokens they move current price and tick to the left or to the right of the current ones,
depending on swap direction. These movements are caused by liquidity being added and removed from pools during swaps.

Pools also track $L$ (`liquidity` variable in our code), which is **the sum of liquidity provided by all price ranges
that include current price**. It's expected that, during big price moves, current price moves outside of price ranges.
When this happens, such price ranges become inactive and their liquidity gets subtracted from $L$. On the other hand,
when current price enters a price range, $L$ is increased and the price range gets activated.

Our current implementation doesn't support such fluidity: we only allow swaps within one active price range. This is what
we're going to fix.

## Updating `computeSwapStep` Function

In `swap` function, we're iterating over initialized ticks (that is, ticks with liquidity) to fill the amount user has
requested. In each iteration, we:

1. find next initialized tick using `tickBitmap.nextInitializedTickWithinOneWord`;
1. swapping in the range between the current price and the next initialized tick (using `SwapMath.computeSwapStep`);
1. always expect that current liquidity is enough to satisfy the swap (i.e. the price after swap is between the current
price and the next initialized tick).

But what happens if the third step is not true? We have this scenario covered in tests:
```solidity
// test/UniswapV3Pool.t.sol
function testSwapBuyEthNotEnoughLiquidity() public {
    ...

    uint256 swapAmount = 5300 ether;

    ...

    vm.expectRevert(stdError.arithmeticError);
    pool.swap(address(this), false, swapAmount, extra);
}
```

The "Arithmetic over/underflow" happens when trying to transfer an amount that's too big (not approved by user). This
happens because, when calculating a swap step, we always expect that next initialized tick always have enough liquidity
to satisfy the swap:

```solidity
// src/lib/SwapMath.sol
function computeSwapStep(...) {
    ...

    sqrtPriceNextX96 = Math.getNextSqrtPriceFromInput(
        sqrtPriceCurrentX96,
        liquidity,
        amountRemaining,
        zeroForOne
    );

    amountIn = ...
    amountOut = ...
}
```

To fix this, we need to pre-calculate the output amount and compare it with the amount requested (`amountRemaining`): if
it's smaller, then the next initialized tick doesn't have enough liquidity. If it's equal to `amountRemaining`, then it's
safe to assume that the next initialized tick has enough liquidity.

Here's how to fix it:
```solidity
// src/lib/SwapMath.sol
function computeSwapStep(...) {
    ...
    amountIn = zeroForOne
        ? Math.calcAmount0Delta(
            sqrtPriceCurrentX96,
            sqrtPriceTargetX96,
            liquidity
        )
        : Math.calcAmount1Delta(
            sqrtPriceCurrentX96,
            sqrtPriceTargetX96,
            liquidity
        );

    if (amountRemaining >= amountIn) sqrtPriceNextX96 = sqrtPriceTargetX96;
    else
        sqrtPriceNextX96 = Math.getNextSqrtPriceFromInput(
            sqrtPriceCurrentX96,
            liquidity,
            amountRemaining,
            zeroForOne
        );

    amountIn = Math.calcAmount0Delta(
        sqrtPriceCurrentX96,
        sqrtPriceNextX96,
        liquidity
    );
    amountOut = Math.calcAmount1Delta(
        sqrtPriceCurrentX96,
        sqrtPriceNextX96,
        liquidity
    );
}
```

We first pre-calculate `amountIn` depending on swap direction. Then, we check if this amount is enough to satisfy the
swap: if so, we're calculating $\sqrt{P}$ of the new price; if not, we're calculating only the amounts that can be provided
by this price range–on the next iteration, we'll try to fill the remaining amount.

I hope this makes sense!

## Updating `swap` Function

Now, in `swap` function, we need to handle the case we introduced in the previous part: when swap price reaches a boundary
of a price range. When this happens, we want to deactivate the price range we're leaving and active the next price range.
We also want to start the next iteration of the loop with the initialized tick we found in this loop.

Here's what we need to add to the end of the loop:
```solidity
if (state.sqrtPriceX96 == step.sqrtPriceNextX96) {
    int128 liquidityDelta = ticks.cross(step.nextTick);

    if (zeroForOne) liquidityDelta = -liquidityDelta;

    state.liquidity = LiquidityMath.addLiquidity(
        state.liquidity,
        liquidityDelta
    );

    if (state.liquidity == 0) revert NotEnoughLiquidity();

    state.tick = zeroForOne ? step.nextTick - 1 : step.nextTick;
} else {
    state.tick = TickMath.getTickAtSqrtRatio(state.sqrtPriceX96);
}
```

The second branch is what we had before–it handles the case when current price stays within the range. So let's focus on
the first one.

`state.sqrtPriceX96` is updated current price, i.e. the price that will be set after the current swap; `step.sqrtPriceNextX96`
is the price at the next initialized tick. If these are equal, we have reached a price range boundary. As discussed above,
when this happens, we want to update $L$ (add or remove liquidity) and continue the swap using the boundary tick as the
current tick.

For convenience, crossing a tick means crossing it from left to right. Thus, lower ticks always add liquidity and upper
ticks always remove it. However, when `zeroForOne` is true, we negate the sign: when price goes down (token $X$ is being
sold), upper ticks add liquidity and lower ticks remove it.

When updating `state.tick`, if price moves down (`zeroForOne` is true), we need to subtract 1 to step out of the price
range. When moving up (`zeroForOne` is false), current tick is always excluded in `TickBitmap.nextInitializedTickWithinOneWord`.

Another small, but very important, change that we need to make is to update $L$ when swapping caused ticks crossing:
```solidity
if (liquidity_ != state.liquidity) liquidity = state.liquidity;
```

We do this after the loop.

## Liquidity Tracking and Ticks Crossing

Let's now look at updated `Tick` library.

First change is in `Tick.Info` structure: we now have two variables to track tick liquidity:
```solidity
struct Info {
    bool initialized;
    // total liquidity at tick
    uint128 liquidityGross;
    // amount of liqudiity added or subtracted when tick is crossed
    int128 liquidityNet;
}
```

`liquidityGross` tracks the absolute liquidity amount of a tick. It's needed to find if tick was flipped or not. `liquidityNet`,
on the other hand, is a signed integer–it tracks the amount of liquidity added (in case of lower tick) or removed
(in case of upper tick) when a tick is crossed.

`liquidityNet` is set in `update` function:
```solidity
function update(
    mapping(int24 => Tick.Info) storage self,
    int24 tick,
    int128 liquidityDelta,
    bool upper
) internal returns (bool flipped) {
    ...

    tickInfo.liquidityNet = upper
        ? int128(int256(tickInfo.liquidityNet) - liquidityDelta)
        : int128(int256(tickInfo.liquidityNet) + liquidityDelta);
}
```

The `cross` function we saw above simply returns `liquidityNet`:
```solidity
function cross(mapping(int24 => Tick.Info) storage self, int24 tick)
    internal
    view
    returns (int128 liquidityDelta)
{
    Tick.Info storage info = self[tick];
    liquidityDelta = info.liquidityNet;
}
```

## Testing

Let's review different liquidity setups and test them to ensure our pool implementation can handle them correctly.

### One Price Range

[TODO: illustrate one price range with current price]

This is the scenario we had earlier. After we have updated the code, we need to ensure old functionality keeps working
correctly.

> For brevity, I'll show only most important parts of the tests. You can find full tests in [the code repo](https://github.com/Jeiwan/uniswapv3-code/blob/milestone_3/test/UniswapV3Pool.Swaps.t.sol).

- When buying ETH:
    ```solidity
    function testBuyETHOnePriceRange() public {
        LiquidityRange[] memory liquidity = new LiquidityRange[](1);
        liquidity[0] = liquidityRange(4545, 5500, 1 ether, 5000 ether, 5000);

        ...

        (int256 expectedAmount0Delta, int256 expectedAmount1Delta) = (
            -0.008396874645169943 ether,
            42 ether
        );

        assertSwapState(
            ExpectedStateAfterSwap({
                ...
                sqrtPriceX96: 5604415652688968742392013927525, // 5003.8180249710795
                tick: 85183,
                currentLiquidity: liquidity[0].amount
            })
        );
    }
    ```
- When buying USDC:
    ```solidity
    function testBuyUSDCOnePriceRange() public {
        LiquidityRange[] memory liquidity = new LiquidityRange[](1);
        liquidity[0] = liquidityRange(4545, 5500, 1 ether, 5000 ether, 5000);

        ...

        (int256 expectedAmount0Delta, int256 expectedAmount1Delta) = (
            0.01337 ether,
            -66.807123823853842027 ether
        );

        assertSwapState(
            ExpectedStateAfterSwap({
                ...
                sqrtPriceX96: 5598737223630966236662554421688, // 4993.683362269102
                tick: 85163,
                currentLiquidity: liquidity[0].amount
            })
        );
    }
    ```

In both of these scenario we buy a small amount of EHT or USDC–it needs to be small enough for the price to not leave
the only price range we created. Key values after swapping is done:
1. `sqrtPriceX96` is slightly above or below the initial price and stays within the price rage;
1. `currentLiquidity` remains unchanged.

> In the theoretical Uniswap V3 introduction, we discussed that $L$ doesn't change when swapping happens within a price
range. We now can see this in practice.

### Multiple Identical and Overlapping Price Ranges

[TODO: illustrate]

- When buying ETH:
    ```solidity
    function testBuyETHTwoEqualPriceRanges() public {
        LiquidityRange memory range = liquidityRange(
            4545,
            5500,
            1 ether,
            5000 ether,
            5000
        );
        LiquidityRange[] memory liquidity = new LiquidityRange[](2);
        liquidity[0] = range;
        liquidity[1] = range;

        ...

        (int256 expectedAmount0Delta, int256 expectedAmount1Delta) = (
            -0.008398516982770993 ether,
            42 ether
        );

        assertSwapState(
            ExpectedStateAfterSwap({
                ...
                sqrtPriceX96: 5603319704133145322707074461607, // 5001.861214026131
                tick: 85179,
                currentLiquidity: liquidity[0].amount + liquidity[1].amount
            })
        );
    }
    ```

- When buying USDC:
    ```solidity
    function testBuyUSDCTwoEqualPriceRanges() public {
        LiquidityRange memory range = liquidityRange(
            4545,
            5500,
            1 ether,
            5000 ether,
            5000
        );
        LiquidityRange[] memory liquidity = new LiquidityRange[](2);
        liquidity[0] = range;
        liquidity[1] = range;
        
        ...

        (int256 expectedAmount0Delta, int256 expectedAmount1Delta) = (
            0.01337 ether,
            -66.827918929906650442 ether
        );

        assertSwapState(
            ExpectedStateAfterSwap({
                ...
                sqrtPriceX96: 5600479946976371527693873969480, // 4996.792621611429
                tick: 85169,
                currentLiquidity: liquidity[0].amount + liquidity[1].amount
            })
        );
    }
    ```

This scenario is similar to the previous one but this time we create two identical price ranges. Two moments are
important here:
1. Price impact is lower than in the previous scenario. The more liquidity the lower the price range.
1. Current liquidity is the sum of the two price ranges. This is because both of them include the current price.

Also, we get slightly more tokens thanks to deeper liquidity.

### Consecutive Price Ranges

[TODO: illustrate both scenarios]

- When buying ETH:
    ```solidity
    function testBuyETHConsecutivePriceRanges() public {
        LiquidityRange[] memory liquidity = new LiquidityRange[](2);
        liquidity[0] = liquidityRange(4545, 5500, 1 ether, 5000 ether, 5000);
        liquidity[1] = liquidityRange(5500, 6250, 1 ether, 5000 ether, 5000);

        ...

        (int256 expectedAmount0Delta, int256 expectedAmount1Delta) = (
            -1.820694594787485635 ether,
            10000 ether
        );

        assertSwapState(
            ExpectedStateAfterSwap({
                ...
                sqrtPriceX96: 6190476002219365604851182401841, // 6105.045728033458
                tick: 87173,
                currentLiquidity: liquidity[1].amount
            })
        );
    }
    ```
- When buying USDC:
    ```solidity
    function testBuyUSDCConsecutivePriceRanges() public {
        LiquidityRange[] memory liquidity = new LiquidityRange[](2);
        liquidity[0] = liquidityRange(4545, 5500, 1 ether, 5000 ether, 5000);
        liquidity[1] = liquidityRange(4000, 4545, 1 ether, 5000 ether, 5000);
        
        ...

        (int256 expectedAmount0Delta, int256 expectedAmount1Delta) = (
            2 ether,
            -9103.264925902176327184 ether
        );

        assertSwapState(
            ExpectedStateAfterSwap({
                ...
                sqrtPriceX96: 5069962753257045266417033265661, // 4094.9666586581643
                tick: 83179,
                currentLiquidity: liquidity[1].amount
            })
        );
    }
    ```

In these scenarios, we make big swaps that cause price to move outside of a price range. As a result, the second price
range gets activated and provides enough liquidity to satisfy the swap. In both scenarios, we can see that price lands
outside of the shorted price range and that the short price range gets deactivated (current liquidity equals to the
liquidity of the second price range).

### Partially Overlapping Price Ranges

[TODO: illustrate both scenarios]

- When buying ETH:
    ```solidity
    function testBuyETHPartiallyOverlappingPriceRanges() public {
        LiquidityRange[] memory liquidity = new LiquidityRange[](2);
        liquidity[0] = liquidityRange(4545, 5500, 1 ether, 5000 ether, 5000);
        liquidity[1] = liquidityRange(5001, 6250, 1 ether, 5000 ether, 5000);
        
        ...

        (int256 expectedAmount0Delta, int256 expectedAmount1Delta) = (
            -1.864220641170389178 ether,
            10000 ether
        );

        assertSwapState(
            ExpectedStateAfterSwap({
                ...
                sqrtPriceX96: 6165345094827913637987008642386, // 6055.578153852725
                tick: 87091,
                currentLiquidity: liquidity[1].amount
            })
        );
    }
    ```

- When buying USDC:
    ```solidity
    function testBuyUSDCPartiallyOverlappingPriceRanges() public {
        LiquidityRange[] memory liquidity = new LiquidityRange[](2);
        liquidity[0] = liquidityRange(4545, 5500, 1 ether, 5000 ether, 5000);
        liquidity[1] = liquidityRange(4000, 4999, 1 ether, 5000 ether, 5000);
        
        ...

        (int256 expectedAmount0Delta, int256 expectedAmount1Delta) = (
            2 ether,
            -9321.077831210790476918 ether
        );

        assertSwapState(
            ExpectedStateAfterSwap({
                ...
                sqrtPriceX96: 5090915820491052794734777344590, // 4128.883835866256
                tick: 83261,
                currentLiquidity: liquidity[1].amount
            })
        );
    }
    ```

This is a variation of the previous scenario, but this time the price ranges are partially overlapping. In this scenario,
I used the same amounts, and, as a result of deeper liquidity in the overlapping area, price impact was lower: we got
more paying less, and the price changes were smaller.