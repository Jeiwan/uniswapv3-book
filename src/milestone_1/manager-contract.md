# Manager Contract

Before deploying our pool contract, we need to solve one problem. As you remember, Uniswap V3 contracts are split into two categories:
1. Core contracts that implement the core functions and don't provide user-friendly interfaces.
2. Periphery contracts that implement user-friendly interfaces for the core contracts.

The pool contract is a core contract, it's not supposed to be user-friendly and flexible. It expects the caller to do all the calculations (prices, amounts) and to provide proper call parameters. It also doesn't use ERC20's `transferFrom` to transfer tokens from the caller. Instead, it uses two callbacks:
1. `uniswapV3MintCallback`, which is called when minting liquidity;
1. `uniswapV3SwapCallback`, which is called when swapping tokens.

In our tests, we implemented these callbacks in the test contract. Since it's only a contract that can implement them, the pool contract cannot be called by regular users (non-contract addresses). This is fine. But not anymore 🙂.

Our next step in the book is deploying the pool contract to a local blockchain and interacting with it from a front-end app. Thus, we need to build a contract that will let non-contract addresses interact with the pool. Let's do this now!

## Workflow

This is how the manager contract will work:
1. To mint liquidity, we'll approve the spending of tokens to the manager contract.
1. We'll then call the `mint` function of the manager contract and pass it minting parameters, as well as the address of the pool we want to provide liquidity into.
1. The manager contract will call the pool's `mint` function and will implement `uniswapV3MintCallback`. It'll have permission to send our tokens to the pool contract.
1. To swap tokens, we'll also approve the spending of tokens to the manager contract.
1. We'll then call the `swap` function of the manager contract and, similarly to minting, it'll pass the call to the pool.
The manager contract will send our tokens to the pool contract, and the pool contract will swap them and send the output amount to us.

Thus, the manager contract will act as an intermediary between users and pools.

## Passing Data to Callbacks

Before implementing the manager contract, we need to upgrade the pool contract.

The manager contract will work with any pool and it'll allow any address to call it. To achieve this, we need to upgrade the callbacks: we want to pass different pool addresses and user addresses to them. Let's look at our current implementation of `uniswapV3MintCallback` (in the test contract):
```solidity
function uniswapV3MintCallback(uint256 amount0, uint256 amount1) public {
    if (transferInMintCallback) {
        token0.transfer(msg.sender, amount0);
        token1.transfer(msg.sender, amount1);
    }
}
```

Key points here:
1. The function transfers tokens belonging to the test contract–we want it to transfer tokens from the caller by using `transferFrom`.
1. The function knows `token0` and `token1`, which will be different for every pool.

Idea: we need to change the arguments of the callback so we can pass user and pool addresses.

Now, let's look at the swap callback:
```solidity
function uniswapV3SwapCallback(int256 amount0, int256 amount1) public {
    if (amount0 > 0 && transferInSwapCallback) {
        token0.transfer(msg.sender, uint256(amount0));
    }

    if (amount1 > 0 && transferInSwapCallback) {
        token1.transfer(msg.sender, uint256(amount1));
    }
}
```

Identically, it transfers tokens from the test contract and it knows `token0` and `token1`.

To pass the extra data to the callbacks, we need to pass it to `mint` and `swap` first (since callbacks are called from these functions). However, since this extra data is not used in the functions and to not make their arguments messier, we'll encode the extra data using [abi.encode()](https://docs.soliditylang.org/en/latest/units-and-global-variables.html?highlight=abi.encode#abi-encoding-and-decoding-functions).

Let's define the extra data as a structure:
```solidity
// src/UniswapV3Pool.sol
...
struct CallbackData {
    address token0;
    address token1;
    address payer;
}
...
```

And then pass encoded data to the callbacks:
```solidity
function mint(
    address owner,
    int24 lowerTick,
    int24 upperTick,
    uint128 amount,
    bytes calldata data // <--- New line
) external returns (uint256 amount0, uint256 amount1) {
    ...
    IUniswapV3MintCallback(msg.sender).uniswapV3MintCallback(
        amount0,
        amount1,
        data // <--- New line
    );
    ...
}

function swap(address recipient, bytes calldata data) // <--- `data` added
    public
    returns (int256 amount0, int256 amount1)
{
    ...
    IUniswapV3SwapCallback(msg.sender).uniswapV3SwapCallback(
        amount0,
        amount1,
        data // <--- New line
    );
    ...
}
```

Now, we can read the extra data in the callbacks in the test contract.
```solidity
function uniswapV3MintCallback(
    uint256 amount0,
    uint256 amount1,
    bytes calldata data
) public {
    if (transferInMintCallback) {
        UniswapV3Pool.CallbackData memory extra = abi.decode(
            data,
            (UniswapV3Pool.CallbackData)
        );

        IERC20(extra.token0).transferFrom(extra.payer, msg.sender, amount0);
        IERC20(extra.token1).transferFrom(extra.payer, msg.sender, amount1);
    }
}
```

Try updating the rest of the code yourself, and if it gets too difficult, feel free to peek [at this commit](https://github.com/Jeiwan/uniswapv3-code/commit/cda23134fd12a190aaeebe718786545621e16c0e).

## Implementing Manager Contract

Besides implementing the callbacks, the manager contract won't do much: it'll simply redirect calls to a pool contract. This is a very minimalistic contract at this moment:
```solidity
pragma solidity ^0.8.14;

import "../src/UniswapV3Pool.sol";
import "../src/interfaces/IERC20.sol";

contract UniswapV3Manager {
    function mint(
        address poolAddress_,
        int24 lowerTick,
        int24 upperTick,
        uint128 liquidity,
        bytes calldata data
    ) public {
        UniswapV3Pool(poolAddress_).mint(
            msg.sender,
            lowerTick,
            upperTick,
            liquidity,
            data
        );
    }

    function swap(address poolAddress_, bytes calldata data) public {
        UniswapV3Pool(poolAddress_).swap(msg.sender, data);
    }

    function uniswapV3MintCallback(...) {...}
    function uniswapV3SwapCallback(...) {...}
}
```

The callbacks are identical to those in the test contract, with the exception that there are no `transferInMintCallback` and `transferInSwapCallback` flags since the manager contract always transfers tokens.

Well, we're now fully prepared to deploy and integrate with a front-end app!