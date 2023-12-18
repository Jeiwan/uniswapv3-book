# Slippage Protection

Slippage is a very important issue in decentralized exchanges. Slippage simply means the difference between the price that you see on the screen when initialing a transaction and the actual price when the swap is executed. This difference appears because there's a short (and sometimes long, depending on network congestion and gas costs) delay between when you send a transaction and when it gets mined. In more technical terms, blockchain state changes every block and there's no guarantee that your transaction will be applied at a specific block.

Another important problem that slippage protection fixes is *sandwich attacks*–this is a common type of attack on decentralized exchange users. During sandwiching, attackers "wrap" your swap transactions in their two transactions: one goes before your transaction and the other goes after it. In the first transaction, an attacker modifies the state of a pool so that your swap becomes very unprofitable for you and somewhat profitable for the attacker. This is achieved by adjusting pool liquidity so that your trade happens at a lower price. In the second transaction, the attacker reestablishes pool liquidity and the price. As a result, you get much fewer tokens than expected due to manipulated prices, and the attacker gets some profit.

![Sandwich attack](images/sandwich_attack.png)

The way slippage protection is implemented in decentralized exchanges is by letting users choose how far the actual price is allowed to drop. By default, Uniswap V3 sets slippage tolerance to 0.1%, which means a swap is executed only if the price at the moment of execution is not smaller than 99.9% of the price the user saw in the browser. This is a very tight range and users are allowed to adjust this number, which is useful when volatility is high.

Let's add slippage protection to our implementation!

## Slippage Protection in Swaps

To protect swaps, we need to add one more parameter to the `swap` function–we want to let the user choose a stop price, a price at which swapping will stop. We'll call the parameter `sqrtPriceLimitX96`:

```solidity
function swap(
    address recipient,
    bool zeroForOne,
    uint256 amountSpecified,
    uint160 sqrtPriceLimitX96,
    bytes calldata data
) public returns (int256 amount0, int256 amount1) {
    ...
    if (
        zeroForOne
            ? sqrtPriceLimitX96 > slot0_.sqrtPriceX96 ||
                sqrtPriceLimitX96 < TickMath.MIN_SQRT_RATIO
            : sqrtPriceLimitX96 < slot0_.sqrtPriceX96 &&
                sqrtPriceLimitX96 > TickMath.MAX_SQRT_RATIO
    ) revert InvalidPriceLimit();
    ...
```

When selling token $x$ (`zeroForOne` is true), `sqrtPriceLimitX96` must be between the current price and the minimal $\sqrt{P}$ since selling token $x$ moves the price down. Likewise, when selling token $y$, `sqrtPriceLimitX96` must be between the current price and the maximal $\sqrt{P}$ because the price moves up.

In the while loop, we want to satisfy two conditions: the full swap amount has not been filled and the current price isn't equal to `sqrtPriceLimitX96`:
```solidity
..
while (
    state.amountSpecifiedRemaining > 0 &&
    state.sqrtPriceX96 != sqrtPriceLimitX96
) {
...
```

This means that Uniswap V3 pools don't fail when slippage tolerance gets hit and simply execute the swap partially.

Another place where we need to consider `sqrtPriceLimitX96` is when calling `SwapMath.computeSwapStep`:

```solidity
(state.sqrtPriceX96, step.amountIn, step.amountOut) = SwapMath
    .computeSwapStep(
        state.sqrtPriceX96,
        (
            zeroForOne
                ? step.sqrtPriceNextX96 < sqrtPriceLimitX96
                : step.sqrtPriceNextX96 > sqrtPriceLimitX96
        )
            ? sqrtPriceLimitX96
            : step.sqrtPriceNextX96,
        state.liquidity,
        state.amountSpecifiedRemaining
    );
```

Here, we want to ensure that `computeSwapStep` never calculates swap amounts outside of `sqrtPriceLimitX96`–this guarantees that the current price will never cross the limiting price.

## Slippage Protection in Minting

Adding liquidity also requires slippage protection. This comes from the fact that price cannot be changed when adding liquidity (liquidity must be proportional to the current price), thus liquidity providers also suffer from slippage. Unlike the `swap` function, however, we're not forced to implement slippage protection in the Pool contract–recall that the Pool contract is a core contract and we don't want to put unnecessary logic into it. This is why we made the Manager contract, and it's in the Manager contract where we'll implement slippage protection.

The Manager contract is a wrapper contract that makes calls to the Pool contract more convenient. To implement slippage protection in the `mint` function, we can simply check the amounts of tokens taken by Pool and compare them to some minimal amounts chosen the by user. Additionally, we can free users from calculating $\sqrt{P_{lower}}$ and $\sqrt{P_{upper}}$, as well as liquidity, and calculate these in `Manager.mint()`.

Our updated `mint` function will now take more parameters, so let's group them in a struct:
```solidity
// src/UniswapV3Manager.sol
contract UniswapV3Manager {
    struct MintParams {
        address poolAddress;
        int24 lowerTick;
        int24 upperTick;
        uint256 amount0Desired;
        uint256 amount1Desired;
        uint256 amount0Min;
        uint256 amount1Min;
    }

    function mint(MintParams calldata params)
        public
        returns (uint256 amount0, uint256 amount1)
    {
        ...
```

`amount0Min` and `amount1Min` are the amounts that are calculated based on slippage tolerance. They must be smaller than the desired amounts, with the gap controlled by the slippage tolerance setting. The liquidity provider expects to provide amounts not smaller than `amount0Min` and `amount1Min`.

Next, we calculate $\sqrt{P_{lower}}$, $\sqrt{P_{upper}}$, and liquidity:
```solidity
...
IUniswapV3Pool pool = IUniswapV3Pool(params.poolAddress);

(uint160 sqrtPriceX96, ) = pool.slot0();
uint160 sqrtPriceLowerX96 = TickMath.getSqrtRatioAtTick(
    params.lowerTick
);
uint160 sqrtPriceUpperX96 = TickMath.getSqrtRatioAtTick(
    params.upperTick
);

uint128 liquidity = LiquidityMath.getLiquidityForAmounts(
    sqrtPriceX96,
    sqrtPriceLowerX96,
    sqrtPriceUpperX96,
    params.amount0Desired,
    params.amount1Desired
);
...
```

`LiquidityMath.getLiquidityForAmounts` is a new function, we'll discuss it in the next chapter.

The next step is to provide liquidity to the pool and check the amounts returned by the pool: if they're too low, we revert.
```solidity
(amount0, amount1) = pool.mint(
    msg.sender,
    params.lowerTick,
    params.upperTick,
    liquidity,
    abi.encode(
        IUniswapV3Pool.CallbackData({
            token0: pool.token0(),
            token1: pool.token1(),
            payer: msg.sender
        })
    )
);

if (amount0 < params.amount0Min || amount1 < params.amount1Min)
    revert SlippageCheckFailed(amount0, amount1);
```

That's it!