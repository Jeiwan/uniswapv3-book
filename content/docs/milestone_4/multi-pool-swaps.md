---
title: "Multi-pool Swaps"
weight: 4
# bookFlatSection: false
# bookToc: true
# bookHidden: false
# bookCollapseSection: false
# bookComments: false
# bookSearchExclude: false
---

# Multi-pool Swaps

We're now proceeding to the core of this milestone–implementing multi-pool swaps in our contracts. We won't touch Pool
contract in this milestone because it's a core contract that should implement only core features. Multi-pool swaps is a
utility feature, and we'll implement it in Manager and Quoter contracts.

## Updating Manager Contract

### Single-pool and Multi-pool Swaps
In our current implementation, `swap` function in Manager contract supports only single-pool swaps and takes pool address
in parameters:

```solidity
function swap(
    address poolAddress_,
    bool zeroForOne,
    uint256 amountSpecified,
    uint160 sqrtPriceLimitX96,
    bytes calldata data
) public returns (int256, int256) { ... }
```

We're going to split it into two functions: single-pool swap and multi-pool swap. These functions will have different
set of parameters:

```solidity
struct SwapSingleParams {
    address tokenIn;
    address tokenOut;
    uint24 tickSpacing;
    uint256 amountIn;
    uint160 sqrtPriceLimitX96;
}

struct SwapParams {
    bytes path;
    address recipient;
    uint256 amountIn;
    uint256 minAmountOut;
}
```

1. `SwapSingleParams` takes pool parameters, input amount, and a limiting price–this is pretty much identical to what
we had before. Notice, that `data` is no longer required.
1. `SwapParams` takes path, output amount recipient, input amount, and minimal output amount. The latter parameter
replaces `sqrtPriceLimitX96` because, when doing multi-pool swaps, we cannot use the slippage protection from Pool
contract (which uses a limiting price). We need to implement another slippage protection, which checks the final output
amount and compares it with `minAmountOut`: the slippage protection fails when the final output amount is smaller than
`minAmountOut`.

### Core Swapping Logic

Let's implement an internal `_swap` function that will be called by both single- and multi-pool swap functions. It'll
prepare parameters and call `Pool.swap`.

```solidity
function _swap(
    uint256 amountIn,
    address recipient,
    uint160 sqrtPriceLimitX96,
    SwapCallbackData memory data
) internal returns (uint256 amountOut) {
    ...
```

`SwapCallbackData` is a new data structure that contains data we pass between swap functions and `uniswapV3SwapCallback`:
```solidity
struct SwapCallbackData {
    bytes path;
    address payer;
}
```

`path` is a swap path and `payer` is the address that provides input tokens in swaps–we'll have different payers during
multi-pool swaps. 

First thing we do in `_swap`, is extracting pool parameters using `Path` library:

```solidity
// function _swap(...) {
(address tokenIn, address tokenOut, uint24 tickSpacing) = data
    .path
    .decodeFirstPool();
```

Then we identify swap direction:

```solidity
bool zeroForOne = tokenIn < tokenOut;
```

Then we make the actual swap:
```solidity
// function _swap(...) {
(int256 amount0, int256 amount1) = getPool(
    tokenIn,
    tokenOut,
    tickSpacing
).swap(
        recipient,
        zeroForOne,
        amountIn,
        sqrtPriceLimitX96 == 0
            ? (
                zeroForOne
                    ? TickMath.MIN_SQRT_RATIO + 1
                    : TickMath.MAX_SQRT_RATIO - 1
            )
            : sqrtPriceLimitX96,
        abi.encode(data)
    );
```

This piece is identical to what we had before but this time we're calling `getPool` to find the pool. `getPool` is
a function that sorts tokens and calls `PoolAddress.computeAddress`:

```solidity
function getPool(
    address token0,
    address token1,
    uint24 tickSpacing
) internal view returns (IUniswapV3Pool pool) {
    (token0, token1) = token0 < token1
        ? (token0, token1)
        : (token1, token0);
    pool = IUniswapV3Pool(
        PoolAddress.computeAddress(factory, token0, token1, tickSpacing)
    );
}
```

After making a swap, we need to figure out which of the amounts is the output one:
```solidity
// function _swap(...) {
amountOut = uint256(-(zeroForOne ? amount1 : amount0));
```

And that's it. Let's now look at how single-pool swap works.

### Single-pool Swapping

`swapSingle` acts simply as a wrapper of `_swap`:

```solidity
function swapSingle(SwapSingleParams calldata params)
    public
    returns (uint256 amountOut)
{
    amountOut = _swap(
        params.amountIn,
        msg.sender,
        params.sqrtPriceLimitX96,
        SwapCallbackData({
            path: abi.encodePacked(
                params.tokenIn,
                params.tickSpacing,
                params.tokenOut
            ),
            payer: msg.sender
        })
    );
}
```

Notice that we're building a one-pool path here: single-pool swap is a multi-pool swap with one pool 🙂.

### Multi-pool Swapping

Multi-pool swapping is only slightly more difficult than single-pool swapping. Let's look at it:

```solidity
function swap(SwapParams memory params) public returns (uint256 amountOut) {
    address payer = msg.sender;
    bool hasMultiplePools;
    ...
```

First swap is paid by user because it's user who provides input tokens.

Then, we start iterating over pools in the path:

```solidity
...
while (true) {
    hasMultiplePools = params.path.hasMultiplePools();

    params.amountIn = _swap(
        params.amountIn,
        hasMultiplePools ? address(this) : params.recipient,
        0,
        SwapCallbackData({
            path: params.path.getFirstPool(),
            payer: payer
        })
    );
    ...
```

In each iteration, we're calling `_swap` with these parameters:
1. `params.amountIn` tracks input amounts. During the first swap it's the amount provided by user. During next swaps
its the amounts returned from previous swaps.
1. `hasMultiplePools ? address(this) : params.recipient`–if there are multiple pools in the path, recipient is the manager
contract, it'll store tokens between swaps. If there's only one pool (last one) in the path, recipient is the one
specified in the parameters (usually the same user that initiates the swap).
1. `sqrtPriceLimitX96` is set to 0 to disable slippage protection in the Pool contract.
1. Last parameter is what we pass to `uniswapV3SwapCallback`–we'll look at it shortly.

After making one swap, we need to proceed to next pool in a path or return:
```solidity
    ...

    if (hasMultiplePools) {
        payer = address(this);
        params.path = params.path.skipToken();
    } else {
        amountOut = params.amountIn;
        break;
    }
}
```

This is where we're changing payer and removing a processed pool from the path.

Finally, the new slippage protection:

```solidity
if (amountOut < params.minAmountOut)
    revert TooLittleReceived(amountOut);
```

### Swap Callback

Let's look at the updated swap callback:

```solidity
function uniswapV3SwapCallback(
    int256 amount0,
    int256 amount1,
    bytes calldata data_
) public {
    SwapCallbackData memory data = abi.decode(data_, (SwapCallbackData));
    (address tokenIn, address tokenOut, ) = data.path.decodeFirstPool();

    bool zeroForOne = tokenIn < tokenOut;

    int256 amount = zeroForOne ? amount0 : amount1;

    if (data.payer == address(this)) {
        IERC20(tokenIn).transfer(msg.sender, uint256(amount));
    } else {
        IERC20(tokenIn).transferFrom(
            data.payer,
            msg.sender,
            uint256(amount)
        );
    }
}
```

The callback expects encoded `SwapCallbackData` with path and payer address. It extracts pool tokens from the path,
figures out swap direction (`zeroForOne`), and the amount the contract needs to transfer out. Then, it acts differently
depending on payer address:
1. If payer is the current contract (this is so when making consecutive swaps), it transfers tokens to the next pool (the
one that called this callback) from current contract's balance.
1. If payer is a different address (the user that initiated the swap), it transfers tokens from user's
balance.

## Updating Quoter Contract

Quoter is another contract that needs to be updated because we want to use it to also find output amounts in multi-pool swaps.
Similarly to Manager, we'll have two variants of `quote` function: single-pool and multi-pool one. Let's look at the
former first.

### Single-pool Quoting
We need to make only a couple of changes in our current `quote` implementation:
1. rename it to `quoteSingle`;
1. extract parameters into a struct (this is mostly a cosmetic change);
1. instead of a pool address, take a token address and a tick spacing in the parameters.

```solidity
// src/UniswapV3Quoter.sol
struct QuoteSingleParams {
    address tokenIn;
    address tokenOut;
    uint24 tickSpacing;
    uint256 amountIn;
    uint160 sqrtPriceLimitX96;
}

function quoteSingle(QuoteSingleParams memory params)
    public
    returns (
        uint256 amountOut,
        uint160 sqrtPriceX96After,
        int24 tickAfter
    )
{
    ...
```

And the only change we have in the body of the function is usage of `getPool` to find pool address:
```solidity
    ...
    IUniswapV3Pool pool = getPool(
        params.tokenIn,
        params.tokenOut,
        params.tickSpacing
    );

    bool zeroForOne = params.tokenIn < params.tokenOut;
    ...
```

### Multi-pool Quoting

Multi-pool quoting implementation is similar to the multi-pool swapping one, but it uses fewer parameters.

```solidity
function quote(bytes memory path, uint256 amountIn)
    public
    returns (
        uint256 amountOut,
        uint160[] memory sqrtPriceX96AfterList,
        int24[] memory tickAfterList
    )
{
    sqrtPriceX96AfterList = new uint160[](path.numPools());
    tickAfterList = new int24[](path.numPools());
    ...
```

As parameters, we only need input amount and swap path. The function returns similar values as `quoteSingle`, but "price
after" and "tick after" are collected after each swap, thus we need to returns arrays.

```solidity
uint256 i = 0;
while (true) {
    (address tokenIn, address tokenOut, uint24 tickSpacing) = path
        .decodeFirstPool();

    (
        uint256 amountOut_,
        uint160 sqrtPriceX96After,
        int24 tickAfter
    ) = quoteSingle(
            QuoteSingleParams({
                tokenIn: tokenIn,
                tokenOut: tokenOut,
                tickSpacing: tickSpacing,
                amountIn: amountIn,
                sqrtPriceLimitX96: 0
            })
        );

    sqrtPriceX96AfterList[i] = sqrtPriceX96After;
    tickAfterList[i] = tickAfter;
    amountIn = amountOut_;
    i++;

    if (path.hasMultiplePools()) {
        path = path.skipToken();
    } else {
        amountOut = amountIn;
        break;
    }
}
```

The logic of the loop is identical to the one in the updated `swap` function:
1. get current pool's parameters;
1. call `quoteSingle` on current pool;
1. save returned values;
1. repeat if there're more pools in the path, or return otherwise.