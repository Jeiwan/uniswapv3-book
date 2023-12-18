# User Interface

We're now ready to update the UI with the changes we made in this milestone. We'll add two new features:
1. Add Liquidity dialog window;
1. slippage tolerance in swapping.


## Add Liquidity Dialog

![Add Liquidity dialog window](images/add_liquidity_dialog.png)

This change will finally remove hard coded liquidity amounts from our code and will allow use to add liquidity at arbitrary ranges.

The dialog is a simple component with a couple of inputs. We can even re-use `addLiquidity` function from previous implementation. However, now we need to convert prices to tick indices in JavaScript: we want users to type in prices but the contracts expect ticks. To make our job easier, we'll use [the official Uniswap V3 SDK](https://github.com/Uniswap/v3-sdk/) for that.

To convert price to $\sqrt{P}$, we can use [encodeSqrtRatioX96](https://github.com/Uniswap/v3-sdk/blob/08a7c050cba00377843497030f502c05982b1c43/src/utils/encodeSqrtRatioX96.ts) function. The function takes two amounts as input and calculates a price by dividing one by the other. Since we only want to convert price to $\sqrt{P}$, we can pass 1 as `amount0`:
```javascript
const priceToSqrtP = (price) => encodeSqrtRatioX96(price, 1);
```

To convert price to tick index, we can use [TickMath.getTickAtSqrtRatio](https://github.com/Uniswap/v3-sdk/blob/08a7c050cba00377843497030f502c05982b1c43/src/utils/tickMath.ts#L82) function. This is an implementation of the Solidity TickMath library in JavaScript:

```javascript
const priceToTick = (price) => TickMath.getTickAtSqrtRatio(priceToSqrtP(price));
```

So we can now convert prices typed in by users to ticks:

```javascript
const lowerTick = priceToTick(lowerPrice);
const upperTick = priceToTick(upperPrice);
```

Another thing we need to add here is slippage protection. For simplicity, I made it a hard coded value and set it to 0.5%. Here's how to use slippage tolerance to calculate minimal amounts:

```javascript
const slippage = 0.5;
const amount0Desired = ethers.utils.parseEther(amount0);
const amount1Desired = ethers.utils.parseEther(amount1);
const amount0Min = amount0Desired.mul((100 - slippage) * 100).div(10000);
const amount1Min = amount1Desired.mul((100 - slippage) * 100).div(10000);
```

## Slippage Tolerance in Swapping

Even though we're the only user of the application and thus will never have problems with slippage during development, let's add an input to control slippage tolerance during swaps.

![Main screen of the web app](images/slippage_tolerance.png)

When swapping, slippage protection is implemented via limiting price–a price we don't to go above or below during a swap. This means that we need to know this price before sending a swap transaction. However, we don't need to calculate it on the front end because Quoter contract does this for us:

```solidity
function quote(QuoteParams memory params)
    public
    returns (
        uint256 amountOut,
        uint160 sqrtPriceX96After,
        int24 tickAfter
    ) { ... }
```

And we're calling Quoter to calculate swap amounts.

So, to calculate limiting price we need to take `sqrtPriceX96After` and subtract slippage tolerance from it–this will be the price we don't want to go below during a swap.

```solidity
const limitPrice = priceAfter.mul((100 - parseFloat(slippage)) * 100).div(10000);
```

And that's it!