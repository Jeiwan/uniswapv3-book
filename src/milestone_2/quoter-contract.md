# Quoter Contract

To integrate our updated Pool contract into the front end app, we need a way to calculate swap amounts without making a swap. Users will type in the amount they want to sell, and we want to calculate and show them the amount they'll get in exchange. We'll do this through Quoter contract.

Since liquidity in Uniswap V3 is scattered over multiple price ranges, we cannot calculate swap amounts with a formula (which was possible in Uniswap V2). The design of Uniswap V3 forces us to use a different approach: to calculate swap amounts, we'll initiate a real swap and will interrupt it in the callback function, grabbing the amounts calculated by Pool contract. That is, we have to simulate a real swap to calculate output amount!

Again, we'll make a helper contract for that:

```solidity
contract UniswapV3Quoter {
    struct QuoteParams {
        address pool;
        uint256 amountIn;
        bool zeroForOne;
    }

    function quote(QuoteParams memory params)
        public
        returns (
            uint256 amountOut,
            uint160 sqrtPriceX96After,
            int24 tickAfter
        )
    {
        ...
```

Quoter is a contract that implements only one public function–`quote`. Quoter is a universal contract that works with any pool so it takes pool address as a parameter. The other parameters (`amountIn` and `zeroForOne`) are required to simulate a swap.

```solidity
try
    IUniswapV3Pool(params.pool).swap(
        address(this),
        params.zeroForOne,
        params.amountIn,
        abi.encode(params.pool)
    )
{} catch (bytes memory reason) {
    return abi.decode(reason, (uint256, uint160, int24));
}
```

The only thing that the contract does is calling `swap` function of a pool. The call is expected to revert (i.e. throw an error)–we'll do this in the swap callback. In the case of a revert, revert reason is decoded and returned; `quote` will never revert. Notice that, in the extra data, we're passing only pool address–in the swap callback, we'll use it to get pool's `slot0` after a swap.

```solidity
function uniswapV3SwapCallback(
    int256 amount0Delta,
    int256 amount1Delta,
    bytes memory data
) external view {
    address pool = abi.decode(data, (address));

    uint256 amountOut = amount0Delta > 0
        ? uint256(-amount1Delta)
        : uint256(-amount0Delta);

    (uint160 sqrtPriceX96After, int24 tickAfter) = IUniswapV3Pool(pool)
        .slot0();
```

In the swap callback, we're collecting values that we need: output amount, new price, and corresponding tick. Next, we need to save these values and revert:

```solidity
assembly {
    let ptr := mload(0x40)
    mstore(ptr, amountOut)
    mstore(add(ptr, 0x20), sqrtPriceX96After)
    mstore(add(ptr, 0x40), tickAfter)
    revert(ptr, 96)
}
```

For gas optimization, this piece is implemented in [Yul](https://docs.soliditylang.org/en/latest/assembly.html), the language used for inline assembly in Solidity. Let's break it down:
1. `mload(0x40)` reads the pointer of the next available memory slot (memory in EVM is organized in 32 byte slots);
1. at that memory slot, `mstore(ptr, amountOut)` writes `amountOut`;
1. `mstore(add(ptr, 0x20), sqrtPriceX96After)` writes `sqrtPriceX96After` right after `amountOut`;
1. `mstore(add(ptr, 0x40), tickAfter)` writes `tickAfter` after `sqrtPriceX96After`;
1. `revert(ptr, 96)` reverts the call and returns 96 bytes (total length of the values we wrote to memory) of data at address `ptr` (start of the data we wrote above).

So, we're basically concatenating the bytes representations of the values we need (exactly what `abi.encode()` does).  Notice that the offsets are always 32 bytes, even though `sqrtPriceX96After` takes 20 bytes (`uint160`) and `tickAfter` takes 3 bytes (`int24`). This is so we could use `abi.decode()` to decode the data: its counterpart, `abi.encode()`, encodes all integers as 32-byte words.

Aaaand...~it's gone~ done.

## Recap

Let's recap to better understand the algorithm:
1. `quote` calls `swap` of a pool with input amount and swap direction;
1. `swap` performs a real swap, it runs the loop to fill the input amount specified by user;
1. to get tokens from user, `swap` calls the swap callback on the caller;
1. the caller (Quote contract) implements the callback, in which it reverts with output amount, new price, and new tick;
1. the revert bubbles up to the initial `quote` call;
1. in `quote`, the revert is caught, revert reason is decoded and returned as the result of calling `quote`.

I hope this is clear!

## Quoter Limitation

This design has one significant limitation: since `quote` calls `swap` function of Pool contract, and `swap` function is not a pure or view function (because it modifies contract state), `quote` cannot also be pure or view. `swap` modifies state and so does `quote`, even if not in Quoter contract. But we treat `quote` as a getter, a function that only reads contract data. This inconsistency means that EVM will use [CALL](https://www.evm.codes/#f1) opcode instead of [STATICCALL](https://www.evm.codes/#fa) when `quote` is called. This is not a big problem since Quoter reverts in the swap callback, and reverting resets the state modified during a call–this guarantees that `quote` won't modify the state of Pool contract (no actual trade will happen).

Another inconvenience that comes from this issue is that calling `quote` from a client library (Ethers.js, Web3.js, etc.) will trigger a transaction. To fix this, we'll need to force the library to make a static call. We'll see how to do this in Ethers.js later in this milestone.
