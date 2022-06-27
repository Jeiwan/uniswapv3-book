---
title: "Providing Liquidity"
weight: 3
# bookFlatSection: false
# bookToc: false
# bookHidden: false
# bookCollapseSection: false
# bookComments: false
# bookSearchExclude: false
---

{{< katex display >}} {{</ katex >}}

# Providing Liquidity

Enough of theory, let's start coding!

Create a new folder (mine is called `uniswapv3-code`), and run `forge init --vscode` in it–this will initialize a Forge
project. The `--vscode` flag tells Forge to configure the Solidity extension for Forge projects.

Next, remove the default contract and its test:
- `script/Contract.s.sol`
- `src/Contract.sol`
- `test/Contract.t.sol`

And that's it! Let's create our first contract!

## Pool Contract

As you've learned from the introduction, Uniswap deploys multiple Pool contracts, each of which is an exchange market of
a pair of tokens. Uniswap groups all its contract into two categories:

- core contracts, and
- periphery contracts.

Core contracts are, as the name implies, the contracts that implement core logic. These are minimal, user-*un*friendly,
low-level contracts. Their purpose is to do one thing. In Uniswap V3, there are 2 such contracts:
1. Pool contract, which implements the core logic of a decentralized exchange.
1. Factory contract, which serves as a registry of Pool contracts and a contract that makes deployment of pools easier.

We'll begin with the pool contract. Create `src/UniswapV3Pool.sol`:

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.14;

contract UniswapV3Pool {}
```

Let's think about what data the contract will store:
1. Since every pool contract is an exchange market of two tokens, we need to track the two token addresses. And these
addresses will be static, set once and forever during contract initialization.
1. Each pool contract is a set of liquidity positions, a data structure to manage positions identified by: liquidity
provider's address, and upper and lower bounds of the position.
1. Each pool contract will also need to maintain a ticks registry and information about each tick–the amount of liquidity
provided by each tick.
1. Since the tick range is limited, we need to store the limits in the contract, as constants.
1. And as we discussed in the introduction, pool contracts store the amount of liquidity, $L$, and $\sqrt{P}$ instead of
token reserves. So we'll need to store them in the contract as well.

Here's what our pool contract with all the state variables:

```solidity
contract UniswapV3Pool {
    using Tick for mapping(int24 => Tick.Info);
    using Position for mapping(bytes32 => Position.Info);
    using Position for Position.Info;

    int24 internal constant MIN_TICK = -887272;
    int24 internal constant MAX_TICK = -MIN_TICK;

    // Pool tokens, immutable
    address public immutable token0;
    address public immutable token1;

    // First slot will contain essential data
    struct Slot0 {
        // Current sqrt(P)
        uint160 sqrtPriceX96;
        // Current tick
        int24 tick;
    }
    Slot0 public slot0;

    // Amount of liquidity, L.
    uint128 public liquidity;

    // Ticks info
    mapping(int24 => Tick.Info) public ticks;
    // Positions info
    mapping(bytes32 => Position.Info) public positions;

    ...
```

Uniswap V3 uses many helper contracts and `Tick` and `Position` are two of them. `using A for B` is a feature of Solidity
that lets you extend type `B` with functions from library contract `A`. This simplifies managing complex data structures.

> For brevity, I'll omit detailed explanation of Solidity syntax and features. Solidity has [great documentation](https://docs.soliditylang.org/en/latest/),
don't hesitate referring to it if something is not clear!

We'll then initialize them in the constructor:

```solidity
    constructor(
        address token0_,
        address token1_,
        uint160 sqrtPriceX96,
        int24 tick
    ) {
        token0 = token0_;
        token1 = token1_;

        slot0 = Slot0({sqrtPriceX96: sqrtPriceX96, tick: tick});
    }
}
```

Notice that we're passing $\sqrt{P}$ and the current tick index without providing liquidity–this sets the current price.

> We're setting both $\sqrt{P}$ and tick index in the constructor for simplicity. Later on, we'll implement the conversion
between $\sqrt{P}$ and tick indexes.

This is our starting point, and our goal in this chapter is to make our first swap.

## Minting

The process of providing liquidity in Uniswap V2 is called *minting*. The reason is that the V2 pool contract mints
tokens (LP-tokens) in exchange for liquidity. V3 doesn't do that, but it still uses the same name for the function. Let's
use it as well:

```solidity
function mint(
    address owner,
    int24 lowerTick,
    int24 upperTick,
    uint128 amount
) external returns (uint256 amount0, uint256 amount1) {
    ...
```

Our `mint` function will take:
1. Owner's address, to track the owner of the liquidity.
1. Upper and lower ticks, to set the bounds of a price range.
1. The amount of liquidity we have provided.

When adding initial liquidity to a pool, this function adds a new tick and a position.

We begin with checking the ticks:
```solidity
if (
    lowerTick >= upperTick ||
    lowerTick < MIN_TICK ||
    upperTick > MAX_TICK
) revert InvalidTickRange();
```

And ensuring that some amount of liquidity is provided:
```solidity
if (amount == 0) revert ZeroLiquidity();
```

Then, add a tick and a position:
```solidity
ticks.update(lowerTick, amount);
ticks.update(upperTick, amount);

Position.Info storage position = positions.get(
    owner,
    lowerTick,
    upperTick
);
position.update(amount);
```

The `ticks.update` function is:
```solidity
// src/libs/Tick.sol
...
function update(
    mapping(int24 => Tick.Info) storage self,
    int24 tick,
    uint128 liquidityDelta
) internal {
    Tick.Info storage tickInfo = self[tick];
    uint128 liquidityBefore = tickInfo.liquidity;
    uint128 liquidityAfter = liquidityBefore + liquidityDelta;

    if (liquidityBefore == 0) {
        tickInfo.initialized = true;
    }

    tickInfo.liquidity = liquidityAfter;
}
...
```
It initialized a tick if it has 0 liquidity before and adds new liquidity to it. As you can see, we're calling this
function on both lower and upper ticks, thus liquidity is added to both of them–we'll see why later on.

The `position.update` function is:
```solidity
// src/libs/Position.sol
function update(Info storage self, uint128 liquidityDelta) internal {
    uint128 liquidityBefore = self.liquidity;
    uint128 liquidityAfter = liquidityBefore + liquidityDelta;

    self.liquidity = liquidityAfter;
}
```
Similar to the tick update function, it adds liquidity to a specific position. And to get a position we call:
```solidity
// src/libs/Position.sol
...
function get(
    mapping(bytes32 => Info) storage self,
    address owner,
    int24 lowerTick,
    int24 upperTick
) internal view returns (Position.Info storage position) {
    position = self[
        keccak256(abi.encodePacked(owner, lowerTick, upperTick))
    ];
}
...
```

Each position is uniquely identified by three keys: owner address, lower tick index, and upper tick index. We're storing
positions in a `bytes32 => Info` map and are using hashes of concatenated owner address, lower tick, and upper tick as
keys. This is cheaper than storing three nested maps.

We're not done yet! Next, we need to calculate the amounts that the user must deposit. Luckily, we have already figured
out the formulas and calculated the exact amounts in the previous part. So, we're going to hardcode the amounts:

```solidity
amount0 = 0.998976618347425280 ether;
amount1 = 5000 ether;
```

> We'll replace these with actual calculations in a later chapter.

Now, we're ready to take tokens from the user. This is done via a callback:
```solidity
uint256 balance0Before;
uint256 balance1Before;
if (amount0 > 0) balance0Before = balance0();
if (amount1 > 0) balance1Before = balance1();
IUniswapV3MintCallback(msg.sender).uniswapV3MintCallback(
    amount0,
    amount1
);
if (amount0 > 0 && balance0Before + amount0 > balance0())
    revert InsufficientInputAmount();
if (amount1 > 0 && balance1Before + amount1 > balance1())
    revert InsufficientInputAmount();
```

First, we record current token balances (if either of them is deposited). Then we call `uniswapV3MintCallback` method
on the caller–this is the callback. It's expected that the caller (whoever executes `mint`) is a contract because
non-contract addresses cannot implement functions in Ethereum. There are two reason doing this like that:

1. Pool contract is a core contract, and core contracts are user-unfriendly. It's expected that core contracts are only
user by other contracts which make interaction with a pool easier.
1. We don't want to calculate the square root of reserves in the contract because it's an expensive operation. But we still
need to be sure that the liquidity deposited by user is correct. To achieve this, we calculate $\Delta x$ and $\Delta y$,
which doesn't require calculating square roots. But this approach forces us to use a callback to let the caller know the
actual amounts they need to deposit.

In production, Pool contracts are called from the Router contract, which handles all the nuances. We'll implement it in
a later chapter.

Finally, we're firing a `Mint` event:
```solidity
emit Mint(msg.sender, owner, lowerTick, upperTick, amount, amount0, amount1);
```

Events is how contract data is indexed in Ethereum for later search. It's a good practice to fire an event whenever
contract's state is changed to let blockchain explorer know when this happened. Events also carry useful information.
In our case it's: caller's address, liquidity position owner's address, upper and lower ticks, new liquidity, and token
amounts. This information will be stored as a log, and anyone else will be able to collect all contract events and
reproduce activity of the contract without traversing and analyzing all blocks and transactions.

And we're done! Phew! Now, let's test minting.

## Testing

At this point we don't know if everything works correctly. Before deploying our contract anywhere we're going to write
a bunch of tests to ensure the contract works correctly. Luckily to us, Forge is a great testing framework and it'll
make testing a breeze. 

Create a new test file:
```solidity
//test/UniswapV3Pool.t.sol
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.14;

import "forge-std/Test.sol";

contract UniswapV3PoolTest is Test {
    function setUp() public {}

    function testExample() public {
        assertTrue(true);
    }
}
```

Let's run it:
```shell
$ forge test
Running 1 test for test/UniswapV3Pool.t.sol:UniswapV3PoolTest
[PASS] testExample() (gas: 279)
Test result: ok. 1 passed; 0 failed; finished in 5.07ms
```

It passes! Of course it is! So far, our test only checks that `true` is `true`!

Test contract are just contract that inherit from `forge-std/Test.sol`. This contract is a set of testing utilities, we'll
get acquainted with them step by step. If you don't want wait, open `lib/forge-std/src/Test.sol` and skim through it!

Test contracts follow a specific convention:
1. `setUp` function is used to set up test cases. In each test cases, we want to have configured environment, like
deployed contracts, minted tokens, initialized pools–we'll do all this in `setUp`.
1. Every test case starts with `test` prefix, e.g. `testMint()`. This will let Forge distinguish test cases from helper
functions (we can add any function we want).

Let's test minting!

### Test Tokens

To test minting we need tokens. This is not a problem because we can deploy any contract in tests! Moreover, Forge can
install open-source contracts as dependencies. Specifically, we need an ERC20 contract with minting functionality. We'll
use the ERC20 contract from [solmate](https://github.com/Rari-Capital/solmate), a collection of gas-optimized contracts
(however, we don't care about gas optimization at this moment) and we'll extend it in a contract that allows public
minting.

Let's install `solmate`:
```shell
$ forge install rari-capital/solmate
```

Then, let's create `ERC20Mintable.sol` contract in `test` folder (we'll use the contract only in tests):
```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.14;

import "solmate/tokens/ERC20.sol";

contract ERC20Mintable is ERC20 {
    constructor(
        string memory _name,
        string memory _symbol,
        uint8 _decimals
    ) ERC20(_name, _symbol, _decimals) {}

    function mint(address to, uint256 amount) public {
        _mint(to, amount);
    }
}
```

Our `ERC20Mintable` inherits all functionality from `solmate/tokens/ERC20.sol` and we additionally implement public `mint`
method which will allow us to mint any number of tokens.

### Minting

Now, we're ready to test minting.

First, let's deploy all the required contracts:
```solidity
// test/UniswapV3Pool.t.sol
...
import "./ERC20Mintable.sol";
import "../src/UniswapV3Pool.sol";

contract UniswapV3PoolTest is Test {
    ERC20Mintable token0;
    ERC20Mintable token1;
    UniswapV3Pool pool;

    function setUp() public {
        token0 = new ERC20Mintable("Ether", "ETH", 18);
        token1 = new ERC20Mintable("USDC", "USDC", 18);
    }

    ...
```
In the `setUp` function, we do things that are required for all our test cases–tokens deployment. We'll need the two
tokens in every future test case (and we'll add more tokens later on). However, we're not going to deploy the pool here
because each test case will need a pool with different parameters.

To make pool setting up cleaner and simpler, we'll do this in a separate function, `setupTestCase`, that takes a set of
test case parameters. In our first test case, we'll test successful liquidity minting. This is what the test case
parameters look like:
```solidity
function testMintSuccess() public {
    TestCaseParams memory params = TestCaseParams({
        wethBalance: 1 ether,
        usdcBalance: 5000 ether,
        currentTick: 85176,
        lowerTick: 84222,
        upperTick: 86129,
        liquidity: 1517882343751509868544,
        currentSqrtP: 5602277097478614198912276234240,
        shouldTransferInCallback: true,
        mintLiqudity: true
    });
    (uint256 poolBalance0, uint256 poolBalance1) = setupTestCase(params);
```
1. We're planning to deposit 1 ETH and 5000 USDC into the pool.
1. We want the current tick to be 85176, and lower and upper ticks being 84222 and 86129 respectively.
1. We're specifying the precalculated liquidity and current $\sqrt{P}$.
1. We also want to deposit liquidity (`mintLiquidity` parameter) and transfer tokens when requested by the pool contract
(`shouldTransferInCallback`). We don't want to do this in each test case, so we want this to be optional.

Next, we're calling `setupTestCase` with the above parameters:
```solidity
function setupTestCase(TestCaseParams memory params)
    internal
    returns (uint256 poolBalance0, uint256 poolBalance1)
{
    token0.mint(address(this), params.wethBalance);
    token1.mint(address(this), params.usdcBalance);

    pool = new UniswapV3Pool(
        address(token0),
        address(token1),
        params.currentSqrtP,
        params.currentTick
    );

    if (params.mintLiqudity) {
        (poolBalance0, poolBalance1) = pool.mint(
            address(this),
            params.lowerTick,
            params.upperTick,
            params.liquidity
        );
    }

    shouldTransferInCallback = params.shouldTransferInCallback;
}
```
In this function, we're minting tokens and deploying a pool (unconditionally), as well as providing liquidity when
`mintLiquidity` is set and setting `shouldTransferInCallback` flag. We'll then check the flag in the mint callback:
```solidity
function uniswapV3MintCallback(uint256 amount0, uint256 amount1) public {
    if (shouldTransferInCallback) {
        token0.transfer(msg.sender, amount0);
        token1.transfer(msg.sender, amount1);
    }
}
```
Recall that test contracts act as users. Thus, this test contract must implement `uniswapV3MintCallback` so it could
call the `mint` function.

Setting up test cases like that is not mandatory, you can do it however feels most comfortable to you. Test contracts are
just contracts. You can implement whatever helper functions you want.

In `testMintSuccess`, we want to test that the pool contract:
1. takes the correct amounts of tokens from us;
1. creates a position with correct key and liquidity;
1. initializes the upper and lower ticks we've specified;
1. has proper current $\sqrt{P}$ and $L$.

Let's do this.

Minting happens in `setupTestCase`, so we don't need to do this again. The function also returns the amounts we have
provided, so let's check them:
```solidity
uint256 expectedAmount0 = 0.998976618347425280 ether;
uint256 expectedAmount1 = 5000 ether;
assertEq(
    poolBalance0,
    expectedAmount0,
    "incorrect token0 deposited amount"
);
assertEq(
    poolBalance1,
    expectedAmount1,
    "incorrect token1 deposited amount"
);
```
We expect specific pre-calculated amounts. And we can also check that these amounts were actually transferred to the pool:
```solidity
assertEq(token0.balanceOf(address(pool)), expectedAmount0);
assertEq(token1.balanceOf(address(pool)), expectedAmount1);
```

Next, we need to check the position the pool created for us. Remember that the key in `positions` mapping is a hash? We
need to calculate it manually and then get our position from the contract:
```solidity
bytes32 positionKey = keccak256(
    abi.encodePacked(address(this), params.lowerTick, params.upperTick)
);
uint128 posLiquidity = pool.positions(positionKey);
assertEq(posLiquidity, params.liquidity);
```

> Since `Position.Info` is a [struct](https://docs.soliditylang.org/en/latest/types.html#structs), it gets destructured
when fetched: each field gets fetched separately.

[TODO: double-check]

Next come the ticks. Again, it's straightforward:
```solidity
(bool tickInitialized, uint128 tickLiquidity) = pool.ticks(
    params.lowerTick
);
assertTrue(tickInitialized);
assertEq(tickLiquidity, params.liquidity);

(tickInitialized, tickLiquidity) = pool.ticks(params.upperTick);
assertTrue(tickInitialized);
assertEq(tickLiquidity, params.liquidity);
```

And finally $\sqrt{P}$ and $L$:
```solidity
(uint160 sqrtPriceX96, int24 tick) = pool.slot0();
assertEq(
    sqrtPriceX96,
    5602277097478614198912276234240,
    "invalid current sqrtP"
);
assertEq(tick, 85176, "invalid current tick");
assertEq(
    pool.liquidity(),
    1517882343751509868544,
    "invalid current liquidity"
);
```

As you can see, writing tests in Solidity is not hard!

### Failures

Of course, testing only successful scenarios is not enough. We also need to test failing cases. What can go wrong when
providing liquidity? Here are a couple of hints:
1. Upper and lower ticks are too big or too low.
1. Zero liquidity is provided.
1. Liquidity provider doesn't have enough of tokens.

I'll leave it for you to implement these scenarios! Feel free peeking at [the code in the repo](https://github.com/Jeiwan/uniswapv3-code/blob/milestone_1/test/UniswapV3Pool.t.sol).
