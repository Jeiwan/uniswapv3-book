# Multi-Pool Swaps

We're now proceeding to the core of this milestone–implementing multi-pool swaps in our contracts. We won't touch the Pool contract in this milestone because it's a core contract that should implement only core features. Multi-pool swaps are a utility feature, and we'll implement it in the Manager and Quoter contracts.

## Updating the Manager Contract

### Single-Pool and Multi-Pool Swaps
In our current implementation, the `swap` function in the Manager contract supports only single-pool swaps and takes pool address in parameters:

```solidity
function swap(
    address poolAddress_,
    bool zeroForOne,
    uint256 amountSpecified,
    uint160 sqrtPriceLimitX96,
    bytes calldata data
) public returns (int256, int256) { ... }
```

We're going to split it into two functions: single-pool swap and multi-pool swap. These functions will have different set of parameters:

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

1. `SwapSingleParams` takes pool parameters, input amount, and a limiting price–this is pretty much identical to what we had before. Notice, that `data` is no longer required.
1. `SwapParams` takes path, output amount recipient, input amount, and minimal output amount. The latter parameter replaces `sqrtPriceLimitX96` because, when doing multi-pool swaps, we cannot use the slippage protection from the Pool contract (which uses a limiting price). We need to implement another slippage protection, which checks the final output amount and compares it with `minAmountOut`: the slippage protection fails when the final output amount is smaller than `minAmountOut`.

### Core Swapping Logic

Let's implement an internal `_swap` function that will be called by both single- and multi-pool swap functions. It'll prepare parameters and call `Pool.swap`.

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

`path` is a swap path and `payer` is the address that provides input tokens in swaps–we'll have different payers during multi-pool swaps. 

The first thing we do in `_swap`, is to extract pool parameters using the `Path` library:

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

This piece is identical to what we had before but this time we're calling `getPool` to find the pool. `getPool` is a function that sorts tokens and calls `PoolAddress.computeAddress`:

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

And that's it. Let's now look at how a single-pool swap works.

### Single-Pool Swapping

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

### Multi-Pool Swapping

Multi-pool swapping is only slightly more difficult than single-pool swapping. Let's look at it:

```solidity
function swap(SwapParams memory params) public returns (uint256 amountOut) {
    address payer = msg.sender;
    bool hasMultiplePools;
    ...
```

The first swap is paid by the user because it's the user who provides input tokens.

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
1. `params.amountIn` tracks input amounts. During the first swap, it's the amount provided by the user. During the next swaps, it's the amounts returned from previous swaps.
1. `hasMultiplePools ? address(this) : params.recipient`–if there are multiple pools in the path, the recipient is the Manager contract, it'll store tokens between swaps. If there's only one pool (the last one) in the path, the recipient is the one specified in the parameters (usually the same user that initiates the swap).
1. `sqrtPriceLimitX96` is set to 0 to disable slippage protection in the Pool contract.
1. The last parameter is what we pass to `uniswapV3SwapCallback`–we'll look at it shortly.

After making one swap, we need to proceed to the next pool in a path or return:
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

The callback expects encoded `SwapCallbackData` with path and payer address. It extracts pool tokens from the path, figures out the swap direction (`zeroForOne`), and the amount the contract needs to transfer out. Then, it acts differently depending on the payer address:
1. If the payer is the current contract (this is so when making consecutive swaps), it transfers tokens to the next pool (the one that called this callback) from the current contract's balance.
1. If the payer is a different address (the user that initiated the swap), it transfers tokens from the user's balance.

## Updating the Quoter Contract

Quoter is another contract that needs to be updated because we want to use it to also find output amounts in multi-pool swaps.  Similarly to Manager, we'll have two variants of the `quote` function: single-pool and multi-pool one. Let's look at the former first.

### Single-pool Quoting
We need to make only a couple of changes in our current `quote` implementation:
1. rename it to `quoteSingle`;
1. extract parameters into a struct (this is mostly a cosmetic change);
1. instead of a pool address, take two token addresses and a tick spacing in the parameters.

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

The only change we have in the body of the function is the usage of `getPool` to find the pool address:
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

As parameters, we only need an input amount and a swap path. The function returns similar values as `quoteSingle`, but "price after" and "tick after" are collected after each swap, thus we need to return arrays.

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
1. get the current pool's parameters;
1. call `quoteSingle` on the current pool;
1. save returned values;
1. repeat if there are more pools in the path, or return otherwise.
