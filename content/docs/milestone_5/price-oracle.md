---
title: "Price Oracle"
weight: 3
# bookFlatSection: false
# bookToc: true
# bookHidden: false
# bookCollapseSection: false
# bookComments: false
# bookSearchExclude: false
---

{{< katex display >}} {{</ katex >}}

# Price Oracle

The final mechanism we're going to add to our DEX is a *price oracle*. Even though it's not essential to a DEX (there
are DEXes that don't implement a price oracle), it's still an important feature of Uniswap and something that's
interesting to learn about.

## What is Price Oracle?

Price oracle is a mechanism that provides asset prices to blockchain. Since blockchains are isolated ecosystems, there's
no direct way of querying external data, e.g. fetching asset prices from centralized exchanges via API. Another, a very
hard one, problem is data validity and authenticity: when fetching prices from an exchange, how do you know they're real?
You have to trust the source. But the internet is not often secure and, sometimes, prices can be manipulated, DNS records
can be hijacked, API servers can go down. All these difficulties need to be addressed so we could have reliable and
correct on-chain prices.

One of the first working solution of the above mentioned problems was [Chainlink](https://chain.link/). Chainlink runs
a decentralized network of oracles that fetch asset prices from centralized exchanges via APIs, average them, and provide
them on-chain in a tamper-proof way. Basically, Chainlink is a set of contracts with one state variable, asset price,
that can be read by anyone (any other contract or user) but can be written to only by oracles.

This is one way of looking at price oracles. There's another.

If we have native on-chain exchanges, why would we need to fetch prices from outside? This how the Uniswap price oracle
works.

Thanks to arbitraging and high liquidity, asset prices on Uniswap are close to those on centralized exchanges. So, instead
of using centralized exchanges as the source of truth for asset prices, we can use Uniswap, and we don't need to solve
the problems related to delivering data on-chain (we also don't need to trust the oracles).

## How Uniswap Price Oracle Works

Uniswap simply keeps the record of all previous swap prices. That's it. But instead of tracking actual prices, Uniswap
tracks the *accumulated price*, which is the sum of prices at each second in the history of a pool contract.

$$a_{i} = \sum_{i=1}^t p_{i}$$

This approach allows to find *time-weighted average price* between two points in time ($t_1$ and $t_2$) by simply
getting the accumulated prices at these points ($a_{t_1}$ and $a_{t_2}$), subtracting one from the other, and dividing
by the number of seconds between the two points:

$$p_{t_1,t_2} = \frac{a_{t_2} - a_{t_1}}{t_2 - t_1}$$

This is how it worked in Uniswap V2. In V3, it's slightly different. The price that's accumulated is the current tick
(which is $log_{1.0001}$ of the price):

$$a_{i} = \sum_{i=1}^t log_{1.0001}P(i)$$

And instead of averaging prices, *geometric mean* is taken:

$$ P_{t_1,t_2} = \left( \prod_{i=t_1}^{t_2} P_i \right) ^ \frac{1}{t_2-t_1} $$

To find the time-weighted geometric mean price between two points in time, we take the accumulated values at these time
points, subtract one from the other, divide by the number of seconds between the two points, and calculate $1.0001^{x}$:

$$ log_{1.0001}{(P_{t_1,t_2})} = \frac{\sum_{i=t_1}^{t_2} log_{1.0001}(P_i)}{t_2-t_1}$$
$$ = \frac{a_{t_2} - a_{t_1}}{t_2-t_1}$$

$$P_{t_1,t_2} = 1.0001^{\frac{a_{t_2} - a_{t_1}}{t_2-t_1}}$$

Uniswap V2 didn't store historical accumulated prices, which required referring to a third-party blockchain data indexing
service to find a historical price when calculating an average one. Uniswap V3, on the other hand, allows to store up
to 65,535 historical accumulated prices, which makes it much easier to calculate any historical time-weighter geometric
price.

## Price Manipulation Mitigation

Last thing worth mentioning discussing is how price manipulation is mitigated in Uniswap.

It's theoretically possible to manipulate a pool's price to your advantage: for example, buy a big amount of tokens to
raise its price and get a profit on a third-party DeFi service that uses Uniswap price oracles, then trade the tokens
back to the real price. To mitigate such attacks, Uniswap tracks prices **at the end of a block**, *after* the last trade
of a block. This removes the possibility of in-block price manipulations.

## Price Oracle Implementation

Alright, let's get to code.

### Observations and Cardinality

We'll begin by creating `Oracle` library contract and `Observation` structure:

```solidity
// src/lib/Oracle.sol
library Oracle {
    struct Observation {
        uint32 timestamp;
        int56 tickCumulative;
        bool initialized;
    }
    ...
}
```

An observation is a fact of recording of a price. A pool contract can store up to 65,535 observations:

```solidity
// src/UniswapV3Pool.sol
contract UniswapV3Pool is IUniswapV3Pool {
    using Oracle for Oracle.Observation[65535];
    ...
    Oracle.Observation[65535] public observations;
}
```

However, since storing that many instances of `Observation` requires a lot of gas (someone would have to pay for writing
each of them to contract's storage), a pool by default can store only 1 observation, which gets overwritten each time.
The number of historical observations, the *cardinality* of observations, can be increased at any time by anyone who's
willing to pay for that. To manage cardinality, we need a few extra state variables:
```solidity
    ...
    struct Slot0 {
        // Current sqrt(P)
        uint160 sqrtPriceX96;
        // Current tick
        int24 tick;
        // Most recent observation index
        uint16 observationIndex;
        // Maximum number of observations
        uint16 observationCardinality;
        // Next maximum number of observations
        uint16 observationCardinalityNext;
    }
    ...
```

`observationIndex` tracks the index of most recent observation; `observationCardinality` tracks the maximum number of
observations the pool can store; `observationCardinalityNext` track the next cardinality the array of observations can
expand to. Observations are stored in a fixed-length array that expands when a new observation is saved and `observationCardinalityNext`
is greater than `observationCardinality` (which signals that cardinality can be expanded). If the array cannot be expanded
(next cardinality value equals to the current one), oldest observations get overwritten, i.e. observation is stored at
index 0, next one is stored at index 1, and so on.

When pool is created, `observationCardinality` and `observationCardinalityNext` are set to 1:
```solidity
// src/UniswapV3Pool.sol
contract UniswapV3Pool is IUniswapV3Pool {
    function initialize(uint160 sqrtPriceX96) public {
        ...

        (uint16 cardinality, uint16 cardinalityNext) = observations.initialize(
            _blockTimestamp()
        );

        slot0 = Slot0({
            sqrtPriceX96: sqrtPriceX96,
            tick: tick,
            observationIndex: 0,
            observationCardinality: cardinality,
            observationCardinalityNext: cardinalityNext
        });
    }
}
```

```solidity
// src/lib/Oracle.sol
library Oracle {
    ...
    function initialize(Observation[65535] storage self, uint32 time)
        internal
        returns (uint16 cardinality, uint16 cardinalityNext)
    {
        self[0] = Observation({
            timestamp: time,
            tickCumulative: 0,
            initialized: true
        });

        cardinality = 1;
        cardinalityNext = 1;
    }
    ...
}
```

### Writing Observations

In `swap` function, when current price is changes, an observation is written to the observations array:

```solidity
// src/UniswapV3Pool.sol
contract UniswapV3Pool is IUniswapV3Pool {
    function swap(
        address recipient,
        bool zeroForOne,
        uint256 amountSpecified,
        uint160 sqrtPriceLimitX96,
        bytes calldata data
    ) public returns (int256 amount0, int256 amount1) {
        ...
        if (state.tick != slot0_.tick) {
            (
                uint16 observationIndex,
                uint16 observationCardinality
            ) = observations.write(
                    slot0_.observationIndex,
                    _blockTimestamp(),
                    slot0_.tick,
                    slot0_.observationCardinality,
                    slot0_.observationCardinalityNext
                );
            
            (
                slot0.sqrtPriceX96,
                slot0.tick,
                slot0.observationIndex,
                slot0.observationCardinality
            ) = (
                state.sqrtPriceX96,
                state.tick,
                observationIndex,
                observationCardinality
            );
        }
        ...
    }
}
```

Notice that the tick that's observed here is **slot0_.tick**, i.e. the price before the swap! It's updated with a new
price in the next statement. When writing a new observation, we first check if there's already an observation at this
block number. If there's one, the new observation is not saved. This is the price manipulation mitigation we discussed
earlier: Uniswap tracks prices **before** the first trade in the block and **after** the last trade in the previous block.

```solidity
// src/lib/Oracle.sol
function write(
    Observation[65535] storage self,
    uint16 index,
    uint32 timestamp,
    int24 tick,
    uint16 cardinality,
    uint16 cardinalityNext
) internal returns (uint16 indexUpdated, uint16 cardinalityUpdated) {
    Observation memory last = self[index];

    if (last.timestamp == timestamp) return (index, cardinality);

    if (cardinalityNext > cardinality && index == (cardinality - 1)) {
        cardinalityUpdated = cardinalityNext;
    } else {
        cardinalityUpdated = cardinality;
    }

    indexUpdated = (index + 1) % cardinalityUpdated;
    self[indexUpdated] = transform(last, timestamp, tick);
}
```

Here we see that an observation is skipped when there's already an observation made at the current block. If there's no
such observation though, we're saving a new one and trying to expand the cardinality when possible. The modulo operator
(`%`) ensures that observation index stays within the range [0, cardinality] and resets to 0 when the upper bound is
reached.

Now, let's look at the `transform` function:

```solidity
function transform(
    Observation memory last,
    uint32 timestamp,
    int24 tick
) internal pure returns (Observation memory) {
    uint56 delta = timestamp - last.timestamp;

    return
        Observation({
            timestamp: timestamp,
            tickCumulative: last.tickCumulative +
                int56(tick) *
                int56(delta),
            initialized: true
        });
}
```

What we're calculating here is the accumulated price: current tick gets multiplied by the number of the seconds since
the last observation and added to the last accumulated price.

### Increase of Cardinality

Let's now see how cardinality is expanded.

Anyone at any time can increase the cardinality of observations of a pool and pay for the gas required to do so. For this,
we'll add a new public function to Pool contract:

```solidity
// src/UniswapV3Pool.sol
function increaseObservationCardinalityNext(
    uint16 observationCardinalityNext
) public {
    uint16 observationCardinalityNextOld = slot0.observationCardinalityNext;
    uint16 observationCardinalityNextNew = observations.grow(
        observationCardinalityNextOld,
        observationCardinalityNext
    );

    if (observationCardinalityNextNew != observationCardinalityNextOld) {
        slot0.observationCardinalityNext = observationCardinalityNextNew;
        emit IncreaseObservationCardinalityNext(
            observationCardinalityNextOld,
            observationCardinalityNextNew
        );
    }
}
```

And a new function to Oracle:

```solidity
// src/lib/Oracle.sol
function grow(
    Observation[65535] storage self,
    uint16 current,
    uint16 next
) internal returns (uint16) {
    if (next <= current) return current;

    for (uint16 i = current; i < next; i++) {
        self[i].timestamp = 1;
    }

    return next;
}
```

In the `grow` function, we're allocating new observations by setting the `timestamp` field of each of them to some non-
zero value. Notice that `self` is a storage variable, assigning values to its elements will update the array counter and
write the values to contract's storage.

### Reading Observations

We've finally come to the trickiest part of this chapter: reading of observations. Before moving on, let's review how
observations are stored to get a better picture.

Observations are stored in a fixed-length array that can be expanded in a separate operation:

[TODO: illustrate]

If a new observation doesn't fit into the array, writing continues starting at index 0, i.e. oldest observations get
overwritten:

[TODO: illustrate]

When reading observations, we want to read them at multiple different historical points, no matter whether there was
an actual observation at a point or not. If there's no observation at a point we need, then we want the contract to
return an average value (the average between two surrounding observations):

[TODO: illustrate]

Also, to keep gas costs low, the contract will return accumulated price, not actual prices: we'll be able to compute
actual prices using the formula above later on.

All of this leads us to this conclusion: to read observations, we're going to find elements in an array, which will
require implementing the [binary search algorithm](https://en.wikipedia.org/wiki/Binary_search_algorithm) to optimize
gas costs.

Let's break it down into smaller steps and begin by implementing `observe` function in `Oracle`:

```solidity
function observe(
    Observation[65535] storage self,
    uint32 time,
    uint32[] memory secondsAgos,
    int24 tick,
    uint16 index,
    uint16 cardinality
) internal view returns (int56[] memory tickCumulatives) {
    tickCumulatives = new int56[](secondsAgos.length);

    for (uint256 i = 0; i < secondsAgos.length; i++) {
        tickCumulatives[i] = observeSingle(
            self,
            time,
            secondsAgos[i],
            tick,
            index,
            cardinality
        );
    }
}
```

The function takes current block timestamp, the list of time points we want to get prices at (`secondsAgo`), current
tick, observations index, and cardinality.

Moving to the `observeSingle` function:

```solidity
function observeSingle(
    Observation[65535] storage self,
    uint32 time,
    uint32 secondsAgo,
    int24 tick,
    uint16 index,
    uint16 cardinality
) internal view returns (int56 tickCumulative) {
    if (secondsAgo == 0) {
        Observation memory last = self[index];
        if (last.timestamp != time) last = transform(last, time, tick);
        return last.tickCumulative;
    }
    ...
}
```

When most recent observation is requested (0 seconds passed), we can return it right away. If it wasn't record in the
current block, transform it to consider the current block and the current tick.

If an older time point is requested, we need several checks before going to the binary search algorithm:
1. if the requested time point is the last observation, we can return the accumulated price at the latest observation;
1. if the requested time point is after the last observation, we can call `transform` to find the accumulated price at
this point, knowing the last observed price and the current price;
1. if the requested time point is before the last observation, we have to use the binary search.

Let's go straight to the third point:
```solidity
function binarySearch(
    Observation[65535] storage self,
    uint32 time,
    uint32 target,
    uint16 index,
    uint16 cardinality
)
    private
    view
    returns (Observation memory beforeOrAt, Observation memory atOrAfter)
{
    ...
```

The function takes the current block timestamp (`time`), the timestamp of the price point requested (`target`), as well
as current observations index and cardinality. It returns the range between two observations in which the requested time
point is located.

To initialize the binary search algorithm, we set the boundaries:
```solidity
uint256 l = (index + 1) % cardinality; // oldest observation
uint256 r = l + cardinality - 1; // newest observation
uint256 i;
```

Recall that the observations array is expected to overflow, that's why we're using the module operator here.

Then we spin up an infinite loop, in which we check the middle point of the range: if it's not initialized (there's no
observation), we're continuing with the next point:

```solidity
while (true) {
    i = (l + r) / 2;

    beforeOrAt = self[i % cardinality];

    if (!beforeOrAt.initialized) {
        l = i + 1;
        continue;
    }

    ...
```

If the point is initialized, we call it the left boundary of the range we want the requested time point to be included
in. And we're trying to find the right boundary (`atOrAfter`):

```solidity
    ...
    atOrAfter = self[(i + 1) % cardinality];

    bool targetAtOrAfter = lte(time, beforeOrAt.timestamp, target);

    if (targetAtOrAfter && lte(time, target, atOrAfter.timestamp))
        break;
    ...
```
If we've found the boundaries, we return them. If not, we continue our search:

```solidity
    ...
    if (!targetAtOrAfter) r = i - 1;
    else l = i + 1;
}
```

After finding a range of observations the requested time point belongs to, we need to calculate the price at the
requested time point:
```solidity
// function observeSingle() {
    ...
    uint56 observationTimeDelta = atOrAfter.timestamp -
        beforeOrAt.timestamp;
    uint56 targetDelta = target - beforeOrAt.timestamp;
    return
        beforeOrAt.tickCumulative +
        ((atOrAfter.tickCumulative - beforeOrAt.tickCumulative) /
            int56(observationTimeDelta)) *
        int56(targetDelta);
    ...
```

This is as simple as finding the average rate of change within the range and multiplying it by the number of seconds
that has passed between the lower bound of the range and the time point we need. (This kind of assumes there's a straight
line between the points of the range and the target time point lies on it.)

The last thing we need to implement here is a public function in Pool contract that reads and returns observations:

```solidity
// src/UniswapV3Pool.sol
function observe(uint32[] calldata secondsAgos)
    public
    view
    returns (int56[] memory tickCumulatives)
{
    return
        observations.observe(
            _blockTimestamp(),
            secondsAgos,
            slot0.tick,
            slot0.observationIndex,
            slot0.observationCardinality
        );
}
```

### Interpreting Observations

Let's now see how to interpret observations.

The `observe` function we just added returns an array of accumulated prices, and we want to know how to convert them
to actual prices. I'll demonstrate this in a test of the `observe` function.

In the test, I run multiple swaps in different directions at different blocks:

```solidity
function testObserve() public {
    ...
    pool.increaseObservationCardinalityNext(3);

    pool.swap(address(this), false, swapAmount, sqrtP(6000), extra);

    vm.warp(7);
    pool.swap(address(this), true, swapAmount2, sqrtP(4000), extra);

    vm.warp(20);
    pool.swap(address(this), false, swapAmount, sqrtP(6000), extra);
    ...
```

> `vm.warp` is a cheat-code provided by Foundry: it forwards to a block with the specified timestamp.

The first swap is make at block with timestamp 1, the second one is made at timestamp 7, and the third one is made at
timestamp 20. We can then read the observations:

```solidity
    ...
    secondsAgos = new uint32[](4);
    secondsAgos[0] = 0;
    secondsAgos[1] = 13;
    secondsAgos[2] = 18;
    secondsAgos[3] = 19;

    int56[] memory tickCumulatives = pool.observe(secondsAgos);
    assertEq(tickCumulatives[0], 1607077);
    assertEq(tickCumulatives[1], 511164);
    assertEq(tickCumulatives[2], 85194);
    assertEq(tickCumulatives[3], 0);
    ...
```

1. The earliest observed price is 0, which is the initial observation that's set when the pool is deployed.
1. After the first swap, tick 85176 was observed (which is the initial price of the pool). However, it wasn't tracked
because the swap was made in the same block where the pool was deployed.
1. Next returned accumulated price is 85194 (~5008 USDC/ETH), which was the price after the first swap. Since we requested
this point 1 second after the observation, the accumulated price is simply `1 * 85194`.
1. Next returned accumulated price is 511164, which is `6 * 85194`–we requested the point 6 seconds after the observation
was made.
1. Finally, the most recent observation is 1607077, which is `(1607077 - 511164) / (20 - 7) = 84301`, which is ~4581
USDC/ETH, the price after the second swap that was observed after the third swap.

I hope this makes sense!

Here's another example, which requests evenly spaced time intervals:

```solidity
secondsAgos = new uint32[](5);
secondsAgos[0] = 0;
secondsAgos[1] = 5;
secondsAgos[2] = 10;
secondsAgos[3] = 15;
secondsAgos[4] = 19;

tickCumulatives = pool.observe(secondsAgos);
assertEq(tickCumulatives[0], 1607077);
assertEq(tickCumulatives[1], 1185572);
assertEq(tickCumulatives[2], 764067);
assertEq(tickCumulatives[3], 340776);
assertEq(tickCumulatives[4], 0);
```

This results in prices: 4581.03, 4581.03, 4747.6, 911.5, which are the average prices within the requested intervals.