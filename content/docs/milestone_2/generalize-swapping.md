---
title: "Generalize Swapping"
weight: 6
# bookFlatSection: false
# bookToc: true
# bookHidden: false
# bookCollapseSection: false
# bookComments: false
# bookSearchExclude: false
---

# Generalize Swapping

This will be the hardest chapter of this milestone. Before updating the code, we need to understand how the algorithm of
swapping in Uniswap V3 works.

You can think of a swap as of filling of an order: a user submits an order to buy a specified amount of tokens from a pool.
The pool will use the available liquidity to "convert" the input amount into an output amount of the other token. If there's
not enough liquidity in the current price range, it'll try to find liquidity in other price ranges (using the function
we implemented in the previous chapter).

We're now going to implement this logic in the `swap` function, however going to stay only within the current price range
for now–we'll implement cross-tick swaps in the next milestone.

```solidity
function swap(
    address recipient,
    bool zeroForOne,
    uint256 amountSpecified,
    bytes calldata data
) public returns (int256 amount0, int256 amount1) {
    ...
```

In `swap` function, we add two new parameters: `zeroForOne` and `amountSpecified`. `zeroForOne` is the flag that controls
swap direction: when `true`, `token0` is traded in for `token1`; when `false,` it's the opposite. For example, if `token0`
is ETH and `token1` is USDC, setting `zeroForOne` to `true` means buying USDC for ETH. `amountSpecified` is the amount of
tokens user wants to sell.

## Filling Orders

Since, in Uniswap V3, liquidity is stored in multiple price ranges, Pool contract needs to find all liquidity that's
required to "fill an order" from user. This is done via iterating over initialized ticks in a direction chosen by
user.

Before continuing, we need to define two new structures:
```solidity
struct SwapState {
    uint256 amountSpecifiedRemaining;
    uint256 amountCalculated;
    uint160 sqrtPriceX96;
    int24 tick;
}

struct StepState {
    uint160 sqrtPriceStartX96;
    int24 nextTick;
    uint160 sqrtPriceNextX96;
    uint256 amountIn;
    uint256 amountOut;
}
```
`SwapState` maintains current swap's state. `amountSpecifiedRemaining` tracks the remaining amount of tokens that needs
to be bought by the pool. When it's zero, the swap is done. `amountCalculated` is the out amount calculated by the contract.
`sqrtPriceX96` and `tick` are new current price and tick after a swap is done.

`StepState` maintains current swap step's state. This structure tracks the state of **one iteration** of an "order filling".
`sqrtPriceStartX96` tracks the price the iteration begins with. `nextTick` is the next initialized tick that will provide
liquidity for the swap and `sqrtPriceNextX96` is the price at the next tick. `amountIn` and `amountOut` are amounts that
can be provided by the liquidity of the current iteration.

> After we implement cross-tick swaps (that is, swaps that happen across multiple price ranges), the idea of iterating
will be clearer.

```solidity
// src/UniswapV3Pool.sol

function swap(...) {
    Slot0 memory slot0_ = slot0;

    SwapState memory state = SwapState({
        amountSpecifiedRemaining: amountSpecified,
        amountCalculated: 0,
        sqrtPriceX96: slot0_.sqrtPriceX96,
        tick: slot0_.tick
    });
    ...
```

Before filling an order, we initialize a `SwapState` instance. We'll loop until `amountSpecifiedRemaining` is 0, which
will mean that the pool has enough liquidity to buy `amountSpecified` tokens from user.

```solidity
...
while (state.amountSpecifiedRemaining > 0) {
    StepState memory step;

    step.sqrtPriceStartX96 = state.sqrtPriceX96;

    (step.nextTick, ) = tickBitmap.nextInitializedTickWithinOneWord(
        state.tick,
        1,
        zeroForOne
    );

    step.sqrtPriceNextX96 = TickMath.getSqrtRatioAtTick(step.nextTick);
```

In the loop, we set up a price range that should provide liquidity for the swap. The range is from `state.sqrtPriceX96`
to `step.sqrtPriceNextX96`, where the latter is the price at the next initialized tick (as returned by
`nextInitializedTickWithinOneWord`–we know this function from a previous chapter).

```solidity
(state.sqrtPriceX96, step.amountIn, step.amountOut) = SwapMath
    .computeSwapStep(
        state.sqrtPriceX96,
        step.sqrtPriceNextX96,
        liquidity,
        state.amountSpecifiedRemaining
    );
```

Next, we're calculating the amounts that can be provider by the current price range, and the new current price the swap
will result in.

```solidity
    state.amountSpecifiedRemaining -= step.amountIn;
    state.amountCalculated += step.amountOut;
    state.tick = TickMath.getTickAtSqrtRatio(state.sqrtPriceX96);
}
```

The final step in the loop is updating the SwapState. `step.amountIn` is the amount of tokens the price range can buy
from user; `step.amountOut` is the related number of the other token the pool can sell to user. `state.sqrtPriceX96` is
the current price that will be set after the swap (recall that trading changes current price).

## SwapMath Contract

Let's look closer at `SwapMath.computeSwapStep`.

```solidity
// src/lib/SwapMath.sol
function computeSwapStep(
    uint160 sqrtPriceCurrentX96,
    uint160 sqrtPriceTargetX96,
    uint128 liquidity,
    uint256 amountRemaining
)
    internal
    pure
    returns (
        uint160 sqrtPriceNextX96,
        uint256 amountIn,
        uint256 amountOut
    )
{
    ...
```

This is the core logic of swapping. The function calculates swap amounts within one price range and respecting available
liquidity. It'll return: the new current price and input and output token amounts. Even though the input amount is provided
by user, we still calculate it to know how much of the user specified input amount was processed by one call to `computeSwapStep`.

```solidity
bool zeroForOne = sqrtPriceCurrentX96 >= sqrtPriceTargetX96;

sqrtPriceNextX96 = Math.getNextSqrtPriceFromInput(
    sqrtPriceCurrentX96,
    liquidity,
    amountRemaining,
    zeroForOne
);
```

By checking the price, we can determine the direction of the swap. Knowing the direction, we can calculate the price after
swapping `amountRemaining` of tokens. We'll return to this function below.

After finding the new price, we can calculate input and output amounts of the swap using the function we already have (
the same functions we used to calculate token amounts from liquidity in the `mint` function):
```solidity
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
```

And swap the amounts if the direction is opposite:
```solidity
if (!zeroForOne) {
    (amountIn, amountOut) = (amountOut, amountIn);
}
```

That's it for `computeSwapStep`!

## Finding Price by Swap Amount

Let's now look at `Math.getNextSqrtPriceFromInput`–the function calculates a $\sqrt{P}$ given another $\sqrt{P}$,
liquidity, and input amount. It tells what the price will be after swapping the specified input amount of tokens, given
the current price and liquidity.

Good news is that we already know the formulas: recall how we calculated `price_next` in Python:
```python
# When amount_in is token0
price_next = int((liq * q96 * sqrtp_cur) // (liq * q96 + amount_in * sqrtp_cur))
# When amount_in is token1
price_next = sqrtp_cur + (amount_in * q96) // liq
```

We're going to implement this in Solidity:
```solidity
// src/lib/Math.sol
function getNextSqrtPriceFromInput(
    uint160 sqrtPriceX96,
    uint128 liquidity,
    uint256 amountIn,
    bool zeroForOne
) internal pure returns (uint160 sqrtPriceNextX96) {
    sqrtPriceNextX96 = zeroForOne
        ? getNextSqrtPriceFromAmount0RoundingUp(
            sqrtPriceX96,
            liquidity,
            amountIn
        )
        : getNextSqrtPriceFromAmount1RoundingDown(
            sqrtPriceX96,
            liquidity,
            amountIn
        );
}
```

The function handles swapping in both directions. Since calculations are different, we'll implement them in separate 
functions.

```solidity
function getNextSqrtPriceFromAmount0RoundingUp(
    uint160 sqrtPriceX96,
    uint128 liquidity,
    uint256 amountIn
) internal pure returns (uint160) {
    uint256 numerator = uint256(liquidity) << FixedPoint96.RESOLUTION;
    uint256 product = amountIn * sqrtPriceX96;

    if (product / amountIn == sqrtPriceX96) {
        uint256 denominator = numerator + product;
        if (denominator >= numerator) {
            return
                uint160(
                    mulDivRoundingUp(numerator, sqrtPriceX96, denominator)
                );
        }
    }

    return
        uint160(
            divRoundingUp(numerator, (numerator / sqrtPriceX96) + amountIn)
        );
}
```
In this function, we're implementing two formulas. At the first `return`, it implements the same formula we implemented
in Python. This is the most precise formula, but it can overflow when multiplying `amountIn` by `sqrtPriceX96`. The
formula is (we discussed it in "Output Amount Calculation"):
$$\sqrt{P_{target}} = \frac{\sqrt{P}L}{\Delta x \sqrt{P} + L}$$

When it overflows, we use an alternative formula, which is less precise:
$$\sqrt{P_{target}} = \frac{L}{\Delta x + \frac{L}{\sqrt{P}}}$$

Which is simply the previous formula with the numerator and the denominator divided by $\sqrt{P}$ to get rid of the multiplication
in the numerator.

The other function has simpler math:
```solidity
function getNextSqrtPriceFromAmount1RoundingDown(
    uint160 sqrtPriceX96,
    uint128 liquidity,
    uint256 amountIn
) internal pure returns (uint160) {
    return
        sqrtPriceX96 +
        uint160((amountIn << FixedPoint96.RESOLUTION) / liquidity);
}
```

## Finishing the Swap

Now, let's return to the `swap` function and finish it.

By this moment, we have looped over next initialized ticks, filled `amountSpecified` specified by user, calculated input
and amount amounts, and found new price and tick. Since, in this milestone, we're implementing only swaps within one price
range, this is enough. We now need to update contract's state, send tokens to user, and get
tokens in exchange.


```solidity
if (state.tick != slot0_.tick) {
    (slot0.sqrtPriceX96, slot0.tick) = (state.sqrtPriceX96, state.tick);
}
```
First, we set new price and tick. Since this operation writes to contract's storage, we want to do it only if the new
tick is different, to optimize gas consumption.

```solidity
(amount0, amount1) = zeroForOne
    ? (
        int256(amountSpecified - state.amountSpecifiedRemaining),
        -int256(state.amountCalculated)
    )
    : (
        -int256(state.amountCalculated),
        int256(amountSpecified - state.amountSpecifiedRemaining)
    );
```

Next, we calculate swap amounts based on swap direction and the amounts calculated during the swap loop.

```solidity
if (zeroForOne) {
    IERC20(token1).transfer(recipient, uint256(-amount1));

    uint256 balance0Before = balance0();
    IUniswapV3SwapCallback(msg.sender).uniswapV3SwapCallback(
        amount0,
        amount1,
        data
    );
    if (balance0Before + uint256(amount0) > balance0())
        revert InsufficientInputAmount();
} else {
    IERC20(token0).transfer(recipient, uint256(-amount0));

    uint256 balance1Before = balance1();
    IUniswapV3SwapCallback(msg.sender).uniswapV3SwapCallback(
        amount0,
        amount1,
        data
    );
    if (balance1Before + uint256(amount1) > balance1())
        revert InsufficientInputAmount();
}
```

Next, we're exchanging tokens with user, depending on swap direction. This piece is identical to what we had in Milestone 2,
only handling of the other swap direction was added.

That's it! Swapping is done!

## Testing

Test won't change significantly, we only need to pass `amountSpecified` and `zeroForOne` to `swap` function. Output amount
will change insignificantly though, because it's now calculated in Solidity.

We can now test swapping in the opposite direction! I'll leave this for you, as a homework (just be sure to choose a
small input amount so the whole swap can be handled by our single price range). Don't hesitate peeking at [my tests](https://github.com/Jeiwan/uniswapv3-code/blob/milestone_2/test/UniswapV3Pool.t.sol)
if this feels difficult!

{{< katex display >}} {{</ katex >}}
