---
title: "First Swap"
weight: 4
# bookFlatSection: false
# bookToc: false
# bookHidden: false
# bookCollapseSection: false
# bookComments: false
# bookSearchExclude: false
---

{{< katex display >}} {{</ katex >}}

# First Swap

Now that we have liquidity, we can make our first swap!

## Calculating Swap Amounts

First step, of course, is to figure out how to calculate swap amounts. And, again, let's pick and hardcode some amount
of USDC we're going to trade in for ETH. Let it be 42! We're going to buy ETH for 42 USDC.

After deciding how many tokens we want to sell, we need to calculate how many tokens we'll get in exchange. There are
multiple ways of doing this. In Uniswap V2, we would've used current pool reserves, but in Uniswap V3 we have $L$ and
$\sqrt{P}$ and we know the fact that, when swapping within a price range, only $\sqrt{P}$ changes and $L$ remains
unchanged. We also know that:
$$L = \frac{\Delta y}{\Delta \sqrt{P}}$$

And... we know $\Delta y$! This is the 42 USDC we're going to trade in! Thus, we can find how selling 42 USDC will affect
the current $\sqrt{P}$ given the $L$:

$$\Delta \sqrt{P} = \frac{\Delta y}{L}$$

In Uniswap V3, we choose **the price we want our trade to lead to** (recall that swapping changes the current price, i.e.
it moves the current price along the curve). Knowing the target price, the contract will calculate the amount of input
token it needs to take from us and the respective amount of output token it'll give us.

Let's plug in our numbers into the above formula:

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

After finding the target price, we can calculate token amounts using the amounts calculation functions from a previous
chapter:

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

Using this formula, we can find the amount of ETH we're buying, $\Delta x$, knowing the price change,
$\Delta\frac{1}{\sqrt{P}}$, and liquidity $L$. Be careful though: $\Delta \frac{1}{\sqrt{P}}$ is not
$\frac{1}{\Delta \sqrt{P}}$! The former is the change of the price of ETH, and it can be found using this expression:

$$\Delta \frac{1}{\sqrt{P}} = \frac{1}{\sqrt{P_{target}}} - \frac{1}{\sqrt{P_{current}}}$$

Luckily, we already know all the values, so we can plug them in right away (this might not fit on your screen!):

$$\Delta \frac{1}{\sqrt{P}} = \frac{1}{5604469350942327889444743441197} - \frac{1}{5602277097478614198912276234240}$$

$$\Delta \frac{1}{\sqrt{P}} = -0.00000553186106731426$$

Now, let's find $\Delta x$:

$$\Delta x = -0.00000553186106731426 * 1517882343751509868544 = -8396714242162698 $$

Which is 0.008396714242162698 ETH, and it's very close to the amount we found above! Notice that this amount is negative
since we're removing it from the pool.