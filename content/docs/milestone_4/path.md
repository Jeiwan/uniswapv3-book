---
title: "Swap Path"
weight: 3
# bookFlatSection: false
# bookToc: true
# bookHidden: false
# bookCollapseSection: false
# bookComments: false
# bookSearchExclude: false
---

# Swap Path

Let's imagine that we have only these pools: WETH/USDC, USDC/USDT, WBTC/USDT. If we want to swap WETH for WBTC, we'll
need to make multiple swaps (WETH→USDC→USDT→WBTC) since there's no WETH/WBTC pool. We can do this manually or we can
improve our contracts to handle such chained, or multi-pool, swaps. Of course, we'll do the latter!

When doing multi-pool swaps, we're sending output of a previous swap to the input of the next one. For example:

1. in WETH/USDC pool, we're selling WETH and buying USDC;
1. in USDC/USDT pool, we're selling USDC from the previous swap and buying USDT;
1. in WBTC/USDT pool, we're selling USDT from the previous pool and buying WBTC.

We can turn this series into a path:

```
WETH/USDC,USDC/USDT,WBTC/USDT
```

And iterate over such path in our contracts to perform multiple swaps in one transaction. However, recall from the previous
chapter that we don't need to know pool addresses and, instead, we can derive them from pool parameters. Thus, the above
swap can be turned into a series of tokens:

```
WETH, USDC, USDT, WBTC
```

And recall that tick spacing is another parameter (besides tokens) that identifies a pool. Thus, the above path becomes:

```
WETH, 60, USDC, 10, USDT, 60, WBTC
```

Where 60 and 10 are tick spacings. We're using 60 in volatile pairs (e.g. ETH/USDC, WBTC/USDT) and 10 in stablecoin
pairs (USDC/USDT).

Now, having such path, we can iterate over it to build pool parameters for each of the pool:

1. `WETH, 60, USDC`;
1. `USDC, 10, USDT`;
1. `USDT, 60, WBTC`.

Knowing these parameters, we can derive pool addresses using `PoolAddress.computeAddress`, which we implemented in the
previous chapter.

> We also can use this concept when doing swaps within one pool: the path would simple contain the parameters of one
pool. And, thus, we can use swap paths in all swaps, universally.

Let's build a library to work with swap paths.

## Path Library

In code, a swap path is a sequence of bytes. In Solidity, a path can be built like that:
```solidity
bytes.concat(
    bytes20(address(weth)),
    bytes3(uint24(60)),
    bytes20(address(usdc)),
    bytes3(uint24(10)),
    bytes20(address(usdt)),
    bytes3(uint24(60)),
    bytes20(address(wbtc))
);
```

And it looks like that:
```shell
0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2 # weth address
  00003c                                   # 60
  A0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48 # usdc address
  00000a                                   # 10
  dAC17F958D2ee523a2206206994597C13D831ec7 # usdt address
  00003c                                   # 60
  2260FAC5E5542a773Aa44fBCfeDf7C193bc2C599 # wbtc address
```

These are the functions that we'll need to implement:
1. calculating the number of pools in a path;
1. figuring out if a path has multiple tokens;
1. extracting first pool parameters from a path;
1. proceeding to the next pair in a path;
1. and decoding first pool parameters.

### Calculating the Number of Pools in a Path
Let's begin with calculating the number of pools in a path:
```solidity
// src/lib/Path.sol
library Path {
    /// @dev The length the bytes encoded address
    uint256 private constant ADDR_SIZE = 20;
    /// @dev The length the bytes encoded tick spacing
    uint256 private constant TICKSPACING_SIZE = 3;

    /// @dev The offset of a single token address + tick spacing
    uint256 private constant NEXT_OFFSET = ADDR_SIZE + TICKSPACING_SIZE;
    /// @dev The offset of an encoded pool key (tokenIn + tick spacing + tokenOut)
    uint256 private constant POP_OFFSET = NEXT_OFFSET + ADDR_SIZE;
    /// @dev The minimum length of a path that contains 2 or more pools;
    uint256 private constant MULTIPLE_POOLS_MIN_LENGTH =
        POP_OFFSET + NEXT_OFFSET;

    ...
```

We first define a few constants:
1. `ADDR_SIZE` is the size of an address, 20 bytes;
1. `TICKSPACING_SIZE` is the size of a tick spacing, 3 bytes (`uint24`);
1. `NEXT_OFFSET` is the offset of a next token address–to get it, we skip an address and a tick spacing;
1. `POP_OFFSET` is the offset of a pool key (token address + tick spacing + token address);
1. `MULTIPLE_POOLS_MIN_LENGTH` is the minimum length of a path that contains 2 or more pools (one set of pool parameters + tick
spacing + token address).

To count the number of pools in a path, we subtract the size of an address (first or last token in a path) and divide
the remaining part by `NEXT_OFFSET` (address + tick spacing):

```solidity
function numPools(bytes memory path) internal pure returns (uint256) {
    return (path.length - ADDR_SIZE) / NEXT_OFFSET;
}
```

### Figuring Out If a Path Has Multiple Tokens
To check if there are multiple pools in a path, we need to compare the length of a path with `MULTIPLE_POOLS_MIN_LENGTH`:

```solidity
function hasMultiplePools(bytes memory path) internal pure returns (bool) {
    return path.length >= MULTIPLE_POOLS_MIN_LENGTH;
}
```

### Extracting First Pool Parameters From a Path

To implement the other functions, we'll need a helper library because Solidity doesn't have native bytes manipulation
functions. Specifically, we'll need a function to extract a sub-array from an array of bytes, and a couple of functions
to convert bytes to `address` and `uint24`.

Luckily, there's a great open-source library called [solidity-bytes-utils](https://github.com/GNSPS/solidity-bytes-utils).
To use the library, we need to extend the `bytes` type in the `Path` library:
```solidity
library Path {
    using BytesLib for bytes;
    ...
}
```

We can implement `getFirstPool` now:
```solidity
function getFirstPool(bytes memory path)
    internal
    pure
    returns (bytes memory)
{
    return path.slice(0, POP_OFFSET);
}
```

The function simply returns the first "token address + tick spacing + token address" segment encoded as bytes.

### Proceeding to a Next Pair in a Path


We'll use the next function when iterating over a path and throwing away processed pools. Notice that we're removing
"token address + tick spacing", not full pool parameters, because we need the other token address to calculate next pool
address.

```solidity
function skipToken(bytes memory path) internal pure returns (bytes memory) {
    return path.slice(NEXT_OFFSET, path.length - NEXT_OFFSET);
}
```

### Decoding First Pool Parameters

And, finally, we need to decode the parameters of the first pool in a path:

```solidity
function decodeFirstPool(bytes memory path)
    internal
    pure
    returns (
        address tokenIn,
        address tokenOut,
        uint24 tickSpacing
    )
{
    tokenIn = path.toAddress(0);
    tickSpacing = path.toUint24(ADDR_SIZE);
    tokenOut = path.toAddress(NEXT_OFFSET);
}
```

Unfortunately, `BytesLib` doesn't implement `toUint24` function but we can implement it ourselves! `BytesLib` has multiple
`toUintXX` functions, so we can take one of them and convert to a `uint24` one:
```solidity
library BytesLibExt {
    function toUint24(bytes memory _bytes, uint256 _start)
        internal
        pure
        returns (uint24)
    {
        require(_bytes.length >= _start + 3, "toUint24_outOfBounds");
        uint24 tempUint;

        assembly {
            tempUint := mload(add(add(_bytes, 0x3), _start))
        }

        return tempUint;
    }
}
```

We're doing this in a new library contract, which we can then use in our Path library alongside `BytesLib`:

```solidity
library Path {
    using BytesLib for bytes;
    using BytesLibExt for bytes;
    ...
}
```
