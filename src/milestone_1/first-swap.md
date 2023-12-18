# First Swap

Now that we have liquidity, we can make our first swap!

## Calculating Swap Amounts

The first step, of course, is to figure out how to calculate swap amounts. And, again, let's pick and hardcode some amount of USDC we're going to trade in for ETH. Let it be 42! We're going to buy ETH for 42 USDC.

After deciding how many tokens we want to sell, we need to calculate how many tokens we'll get in exchange. In Uniswap V2, we would've used current pool reserves, but in Uniswap V3 we have $L$ and $\sqrt{P}$ and we know the fact that when swapping within a price range, only $\sqrt{P}$ changes and $L$ remains unchanged (Uniswap V3 acts exactly as V2 when swapping is done only within one price range). We also know that:

$$L = \frac{\Delta y}{\Delta \sqrt{P}}$$

And... we know $\Delta y$! This is the 42 USDC we're going to trade in! Thus, we can find how selling 42 USDC will affect the current $\sqrt{P}$ given the $L$:

$$\Delta \sqrt{P} = \frac{\Delta y}{L}$$

In Uniswap V3, we choose **the price we want our trade to lead to** (recall that swapping changes the current price, i.e. it moves the current price along the curve). Knowing the target price, the contract will calculate the amount of input token it needs to take from us and the respective amount of output token it'll give us.

Let's plug our numbers into the above formula:

$$\Delta \sqrt{P} = \frac{42 \enspace USDC}{1517882343751509868544} = 2192253463713690532467206957$$

After adding this to the current $\sqrt{P}$, we'll get the target price:

$$\sqrt{P_{target}} = \sqrt{P_{current}} + \Delta \sqrt{P}$$

$$\sqrt{P_{target}} = 5604469350942327889444743441197$$

> To calculate the target price in Python:
> ```python
> amount_in = 42 * eth
> price_diff = (amount_in * q96) // liq
> price_next = sqrtp_cur + price_diff
> print("New price:", (price_next / q96) ** 2)
> print("New sqrtP:", price_next)
> print("New tick:", price_to_tick((price_next / q96) ** 2))
> # New price: 5003.913912782393
> # New sqrtP: 5604469350942327889444743441197
> # New tick: 85184
> ```

After finding the target price, we can calculate token amounts using the amounts calculation functions from a previous chapter:

$$ x = \frac{L(\sqrt{p_b}-\sqrt{p_a})}{\sqrt{p_b}\sqrt{p_a}}$$
$$ y = L(\sqrt{p_b}-\sqrt{p_a}) $$

> In Python:
> ```python
> amount_in = calc_amount1(liq, price_next, sqrtp_cur)
> amount_out = calc_amount0(liq, price_next, sqrtp_cur)
> 
> print("USDC in:", amount_in / eth)
> print("ETH out:", amount_out / eth)
> # USDC in: 42.0
> # ETH out: 0.008396714242162444
> ```

To verify the amounts, let's recall another formula:

$$\Delta x = \Delta \frac{1}{\sqrt{P}} L$$

Using this formula, we can find the amount of ETH we're buying, $\Delta x$, knowing the price change, $\Delta\frac{1}{\sqrt{P}}$, and liquidity $L$. Be careful though: $\Delta \frac{1}{\sqrt{P}}$ is not $\frac{1}{\Delta \sqrt{P}}$! The former is the change in the price of ETH, and it can be found using this expression:

$$\Delta \frac{1}{\sqrt{P}} = \frac{1}{\sqrt{P_{target}}} - \frac{1}{\sqrt{P_{current}}}$$

Luckily, we already know all the values, so we can plug them in right away (this might not fit on your screen!):

$$\Delta \frac{1}{\sqrt{P}} = \frac{1}{5604469350942327889444743441197} - \frac{1}{5602277097478614198912276234240}$$

$$= -6.982190286589445\text{e-}35 * 2^{96} $$
$$= -0.00000553186106731426$$

Now, let's find $\Delta x$:

$$\Delta x = -0.00000553186106731426 * 1517882343751509868544 = -8396714242162698 $$

Which is 0.008396714242162698 ETH, and it's very close to the amount we found above! Notice that this amount is negative since we're removing it from the pool.

## Implementing a Swap

Swapping is implemented in the `swap` function:
```solidity
function swap(address recipient)
    public
    returns (int256 amount0, int256 amount1)
{
    ...
```
At this moment, it only takes a recipient, who is a receiver of tokens.

First, we need to find the target price and tick, as well as calculate the token amounts. Again, we'll simply hard-code the values we calculated earlier to keep things as simple as possible:
```solidity
...
int24 nextTick = 85184;
uint160 nextPrice = 5604469350942327889444743441197;

amount0 = -0.008396714242162444 ether;
amount1 = 42 ether;
...
```

Next, we need to update the current tick and `sqrtP` since trading affects the current price:
```solidity
...
(slot0.tick, slot0.sqrtPriceX96) = (nextTick, nextPrice);
...
```

Next, the contract sends tokens to the recipient and lets the caller transfer the input amount into the contract:
```solidity
...
IERC20(token0).transfer(recipient, uint256(-amount0));

uint256 balance1Before = balance1();
IUniswapV3SwapCallback(msg.sender).uniswapV3SwapCallback(
    amount0,
    amount1
);
if (balance1Before + uint256(amount1) < balance1())
    revert InsufficientInputAmount();
...
```

Again, we're using a callback to pass the control to the caller and let it transfer the tokens. After that, we check that the pool's balance is correct and includes the input amount.

Finally, the contract emits a `Swap` event to make the swap discoverable. The event includes all the information about the swap:
```solidity
...
emit Swap(
    msg.sender,
    recipient,
    amount0,
    amount1,
    slot0.sqrtPriceX96,
    liquidity,
    slot0.tick
);
```

And that's it! The function simply sends some amount of tokens to the specified recipient address and expects a certain number of the other tokens in exchange. Throughout this book, the function will get much more complicated.

## Testing Swapping

Now, we can test the swap function. In the same test file, create the `testSwapBuyEth` function and set up the test case. This test case uses the same parameters as `testMintSuccess`:
```solidity
function testSwapBuyEth() public {
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

    ...
```

The next steps will be different, however.

> We're not going to test that liquidity has been correctly added to the pool since we tested this functionality in the other test cases.

To make the test swap, we need 42 USDC:
```solidity
token1.mint(address(this), 42 ether);
```

Before making the swap, we need to ensure we can transfer tokens to the pool contract when it requests them:
```solidity
function uniswapV3SwapCallback(int256 amount0, int256 amount1) public {
    if (amount0 > 0) {
        token0.transfer(msg.sender, uint256(amount0));
    }

    if (amount1 > 0) {
        token1.transfer(msg.sender, uint256(amount1));
    }
}
```
Since amounts during a swap can be positive (the amount that's sent to the pool) and negative (the amount that's taken from the pool), in the callback, we only want to send the positive amount, i.e. the amount we're trading in.

Now, we can call `swap`:
```solidity
(int256 amount0Delta, int256 amount1Delta) = pool.swap(address(this));
```

The function returns token amounts used in the swap, and we can check them right away:
```solidity
assertEq(amount0Delta, -0.008396714242162444 ether, "invalid ETH out");
assertEq(amount1Delta, 42 ether, "invalid USDC in");
```

Then, we need to ensure that tokens were transferred from the caller:
```solidity
assertEq(
    token0.balanceOf(address(this)),
    uint256(userBalance0Before - amount0Delta),
    "invalid user ETH balance"
);
assertEq(
    token1.balanceOf(address(this)),
    0,
    "invalid user USDC balance"
);
```

And sent to the pool contract:
```solidity
assertEq(
    token0.balanceOf(address(pool)),
    uint256(int256(poolBalance0) + amount0Delta),
    "invalid pool ETH balance"
);
assertEq(
    token1.balanceOf(address(pool)),
    uint256(int256(poolBalance1) + amount1Delta),
    "invalid pool USDC balance"
);
```

Finally, we're checking that the pool state was updated correctly:
```solidity
(uint160 sqrtPriceX96, int24 tick) = pool.slot0();
assertEq(
    sqrtPriceX96,
    5604469350942327889444743441197,
    "invalid current sqrtP"
);
assertEq(tick, 85184, "invalid current tick");
assertEq(
    pool.liquidity(),
    1517882343751509868544,
    "invalid current liquidity"
);
```

Notice that swapping doesn't change the current liquidity–in a later chapter, we'll see when it does change it.

## Homework

Write a test that fails with an `InsufficientInputAmount` error. Keep in mind that there's a hidden bug 🙂