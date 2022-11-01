---
title: "Factory Contract"
weight: 2
# bookFlatSection: false
# bookToc: true
# bookHidden: false
# bookCollapseSection: false
# bookComments: false
# bookSearchExclude: false
---

# Factory Contract

Uniswap is designed in a way that assumes many discrete Pool contracts, with each pool handling swaps of one token pair.
This looks problematic when we want to swap between two tokens that don't have a pool–if there's no pool, no swaps are
possible. However, we can still do intermediate swaps: first swap to a token that has pairs with either of the tokens
and then swap this token to the target token. This can also go deeper and have more intermediate tokens. However,
doing this manually is cumbersome, and, luckily, we can make the process easier by implementing it in our smart contracts.

*Factory* contract is a contract that serves multiple purposes:
1. It acts as a centralized registry of Pool contracts. Using a factory, you can find all deployed pools, their tokens,
and addresses.
1. It simplifies deployment of Pool contracts. EVM allows to deploy smart contracts from smart contracts–Factory uses
this feature to make pools deployment a breeze.
1. It makes pool addresses predictable and allows to compute them without making calls to the registry. This makes
pools easily discoverable.

Let's build Factory contract! But before doing this, we need to learn something new.

## `CREATE` and `CREATE2` Opcodes

EVM has two ways of deploying contracts: via `CREATE` or via `CREATE2` opcode. The only difference between them is how
new contract address is generated:
1. `CREATE` uses deployer's account `nonce` to generate a contract address (in pseudocode):
    ```
    KECCAK256(deployer.address, deployer.nonce)
    ```
    `nonce` is an account-specific counter of transactions. Using `nonce` in new contract address generation makes it hard
    to compute an address in other contracts or off-chain apps, mainly because, to find the nonce a contract was deployed at,
    one needs to scan historical account transactions.
1. `CREATE2` uses a custom *salt* to generate a contract address. This is just an arbitrary sequence of bytes chosen
by a developer, which is used to make address generation deterministic (and reduces the chance of a collision).
    ```
    KECCAK256(deployer.address, salt, contractCodeHash)
    ```

We need to know the difference because Factory uses `CREATE2` when deploying Pool contracts so pools get unique and
deterministic addresses that can be computed in other contracts and off-chain apps. Specifically, for salt, Factory
computes a hash using these pool parameters:
```solidity
keccak256(abi.encodePacked(token0, token1, tickSpacing))
```

`token0` and `token` are the addresses of pool tokens, and `tickSpacing` is something we're going to learn about next.

## Tick Spacing

Recall the loop in `swap` function:
```solidity
while (
    state.amountSpecifiedRemaining > 0 &&
    state.sqrtPriceX96 != sqrtPriceLimitX96
) {
    ...
    (step.nextTick, ) = tickBitmap.nextInitializedTickWithinOneWord(...);
    (state.sqrtPriceX96, step.amountIn, step.amountOut) = SwapMath.computeSwapStep(...);
    ...
}
```

This loop finds initialized ticks that have some liquidity by iterating them in either of the directions. This iterating,
however, is an expensive operation: if a tick is far away, the code would need to pass all the ticks between the current
and the target one, which consumes gas. To make this loop more gas-efficient, Uniswap pools have `tickSpacing` setting,
which sets, as the name suggest, the distance between ticks: the wider the distance, the more gas efficient swaps are.

However, the wider a tick spacing the lower the precision. Low volatility pairs (e.g. stablecoin pairs) need higher
precision because price movements are narrow in such pairs. Medium and high volatility pairs need lower precision since
price movement are wide in such pairs. To handle this diversity, Uniswap allows to pick a tick spacing when a pair is
deployed. Uniswap allows deployers to choose from these options: 10, 60, or 200. And we'll have only 10 and 60 for
simplicity.

In technical terms, tick indexes can only be multiples of `tickSpacing`: if `tickSpacing` is 10, only multiples of 10
will be valid as tick indexes (10, 20, 5000, 5010, but not 8, 12, 5001, etc.). However, and this is important, this doesn't
apply to the current price–it can still be any tick because we want it to be as precise as possible. `tickSpacing` is
only applied to price ranges.

Thus, each pool is uniquely identified by this set of parameters:
1. `token0`,
1. `token1`,
1. `tickSpacing`;

> And, yes, there can be pools with the same tokens but different tick spacings.

Factory contract uses this set of parameters as a unique identifier of a pool and passes it as a salt to generate a new
pool contract address.

> From now on, we'll assume the tick spacing of 60 for all our pools, and we'll use 10 for stablecoin pairs.

## Factory Implementation

In the constructor of Factory, we need to initialize supported tick spacings:
```solidity
// src/UniswapV3Factory.sol
contract UniswapV3Factory is IUniswapV3PoolDeployer {
    mapping(uint24 => bool) public tickSpacings;
    constructor() {
        tickSpacings[10] = true;
        tickSpacings[60] = true;
    }

    ...
```

> We could've made them constants, but we'll need to have it as a mapping for a later milestone (tick spacings will
have different swap fee amounts).

Factory contract is a contract with only one function `createPool`. The function begins with necessary checks we need to
make before creating a pool:
```solidity
// src/UniswapV3Factory.sol
contract UniswapV3Factory is IUniswapV3PoolDeployer {
    PoolParameters public parameters;
    mapping(address => mapping(address => mapping(uint24 => address)))
        public pools;

    ...

    function createPool(
        address tokenX,
        address tokenY,
        uint24 tickSpacing
    ) public returns (address pool) {
        if (tokenX == tokenY) revert TokensMustBeDifferent();
        if (!tickSpacings[tickSpacing]) revert UnsupportedTickSpacing();

        (tokenX, tokenY) = tokenX < tokenY
            ? (tokenX, tokenY)
            : (tokenY, tokenX);

        if (tokenX == address(0)) revert TokenXCannotBeZero();
        if (pools[tokenX][tokenY][tickSpacing] != address(0))
            revert PoolAlreadyExists();
        
        ...
```

Notice that this is first time when we're sorting tokens:
```solidity
(tokenX, tokenY) = tokenX < tokenY
    ? (tokenX, tokenY)
    : (tokenY, tokenX);
```
From now on, we'll also expect pool token addresses to be sorted, i.e. `token0` goes before `token1` when sorted. We'll
enforce this to make salt (and pool addresses) computation consistent.

> This change also affects how we deploy tokens in tests and the deployment script: we need to ensure that WETH is always
`token0` to make price calculations simpler in Solidity (otherwise, we'd need to use fractional prices, like 1/5000). If
WETH is not `token0` in your tests, change the order of token deployments.

After that, we prepare pool parameters and deploy a pool:
```solidity
parameters = PoolParameters({
    factory: address(this),
    token0: tokenX,
    token1: tokenY,
    tickSpacing: tickSpacing
});

pool = address(
    new UniswapV3Pool{
        salt: keccak256(abi.encodePacked(tokenX, tokenY, tickSpacing))
    }()
);

delete parameters;
```

This piece looks weird because `parameters` is not used. Uniswap uses [Inversion of Control](https://en.wikipedia.org/wiki/Inversion_of_control)
to pass parameters to a pool during deployment. Let's look at updated Pool contract constructor:
```solidity
// src/UniswapV3Pool.sol
contract UniswapV3Pool is IUniswapV3Pool {
    ...
    constructor() {
        (factory, token0, token1, tickSpacing) = IUniswapV3PoolDeployer(
            msg.sender
        ).parameters();
    }
    ..
}
```

Aha! Pool expects its deployer to implement `IUniswapV3PoolDeployer` interface (which only defines the `parameters()` getter)
and calls it in the constructor during deployment to get the parameters. This is what the flow looks like:
1. `Factory`: defines `parameters` state variable (implements `IUniswapV3PoolDeployer`) and sets it before deploying a
pool.
1. `Factory`: deploys a pool.
1. `Pool`: in the constructor, calls `parameters()` function on its deployer and expects that pool parameters are
returned.
1. `Factory`: calls `delete parameters;` to clean up the slot of `parameters` state variable and to reduce gas consumption.
This is a temporary state variable that has a value only during a call to `createPool()`.

After a pool is created, we keep it in the `pools` mapping (so it can be found by its tokens) and emit an event:
```solidity
    pools[tokenX][tokenY][tickSpacing] = pool;
    pools[tokenY][tokenX][tickSpacing] = pool;

    emit PoolCreated(tokenX, tokenY, tickSpacing, pool);
}
```

## Pool Initialization

As you have noticed from the code above, we no longer set `sqrtPriceX96` and `tick` in Pool's constructor–this is now
done in a separate function, `initialize`, that needs to be called after pool is deployed:

```solidity
// src/UniswapV3Pool.sol
function initialize(uint160 sqrtPriceX96) public {
    if (slot0.sqrtPriceX96 != 0) revert AlreadyInitialized();

    int24 tick = TickMath.getTickAtSqrtRatio(sqrtPriceX96);

    slot0 = Slot0({sqrtPriceX96: sqrtPriceX96, tick: tick});
}
```

So this is how we deploy pools now:

```solidity
UniswapV3Factory factory = new UniswapV3Factory();
UniswapV3Pool pool = UniswapV3Pool(factory.createPool(token0, token1, tickSpacing));
pool.initialize(sqrtP(currentPrice));
```

## `PoolAddress` Library

Let's now implement a library that will help us calculate pool contract addresses from other contracts. This library
will have only one function, `computeAddress`:
```solidity
// src/lib/PoolAddress.sol
library PoolAddress {
    function computeAddress(
        address factory,
        address token0,
        address token1,
        uint24 tickSpacing
    ) internal pure returns (address pool) {
        require(token0 < token1);
        ...
```

The function needs to know pool parameters (they're used to build a salt) and Factory contract address. It expects the
tokens to be sorted, which we discussed above.

Now, the core of the function:
```solidity
pool = address(
    uint160(
        uint256(
            keccak256(
                abi.encodePacked(
                    hex"ff",
                    factory,
                    keccak256(
                        abi.encodePacked(token0, token1, tickSpacing)
                    ),
                    keccak256(type(UniswapV3Pool).creationCode)
                )
            )
        )
    )
);
```

This is what `CREATE2` does under the hood to calculate new token address. Let's unwind it:

1. first, we calculate salt (`abi.encodePacked(token0, token1, tickSpacing)`) and hash it;
1. then, we obtain Pool contract code (`type(UniswapV3Pool).creationCode`) and also hash it;
1. then, we build a sequence of bytes that includes: `0xff`, Factory contract address, hashed salt, and hashed Pool
contract code;
1. we then hash the sequence and convert it to an address.

These steps implement contract address generation as it's defined in [EIP-1014](https://eips.ethereum.org/EIPS/eip-1014),
which is the EIP that added `CREATE2` opcode. Let's look closer at the values that constitute the hashed byte sequence:
1. `0xff`, as defined in the EIP, is used to distinguish addresses generated by `CREATE` and `CREATE2`;
1. `factory` is the address of the deployer, in our case a Factory contract;
1. salt was discussed earlier–it uniquely identifies a pool;
1. hashed contract code is needed to protect from collisions: different contracts can have the same salt, but their code
hash will be different.

So, according to this scheme, a contract address is a hash of the values that uniquely identify this contract, including
its deployer, code, and unique parameters. We can use this function from anywhere to find out a pool address without
making any external calls and without querying the factory.

## Simplified Interfaces of Manager and Quoter

In Manager and Quoter contracts, we no longer need to ask users for pool address! This makes interaction with the contracts
easier because users don't need to know pool addresses, they only need to know tokens. However, users also need to specify
tick spacing because it's included in pool's salt.

Moreover, we no longer need to ask users for the `zeroForOne` flag because we can now always figure it out thanks to
tokens sorting. `zeroForOne` is true when "from token" is less than "to token", since pool's `token0` is always less than
`token1`. Likewise, `zeroForOne` is always false when "from token" is greater than "to token".

> Addresses are hashes, and hashes are numbers, so we can say "less than" or "greater that" when comparing addresses.
