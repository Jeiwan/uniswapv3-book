---
title: "Calculating Liquidity"
weight: 2
# bookFlatSection: false
# bookToc: true
# bookHidden: false
# bookCollapseSection: false
# bookComments: false
# bookSearchExclude: false
---

# Calculating liquidity

Trading is not possible without liquidity, and to make our first swap we need to put some liquidity into the pool contract.
Here's what we need to know to add liquidity to the pool contract:

1. A price range. As a liquidity provider, we want to provide liquidity at a specific price range, and it'll only be used
in this range.
1. Amount of liquidity, which is the amounts of two tokens. We'll need to transfer these amounts to the pool contract.

Here, we're going to calculate these manually, but, in a later chapter, a contract will do this for us. Let's begin with
a price range.

## Price Range Calculation

Recall that, in Uniswap V3, the entire price range is demaracted into *ticks*: each tick corresponds to a price and has
an index. In our first pool implementation, we're going to buy ETH for USDC at the price of <span>$5000</span> per 1 ETH.
Buying ETH will remove some amount of it from the pool and will push the price slightly above <span>$5000</span>.
We want to provide liquidity at a range that includes this price. And we want to be sure that the final price will stay
**within this range** (we'll do multi-range swaps in a later chapter).

We'll need to find three ticks:
1. The current tick will correspond to the current price (5000 USDC for 1 ETH).
1. The lower and upper bounds of the price range we're providing liquidity into. Let the lower price be <span>$4500</span>
and the upper price be <span>$5500</span>.

From the theoretical introduction we know that:

$$\sqrt{p} = \sqrt{\frac{y}{x}}$$

Since we've agreed to use ETH as the $x$ reserve and USDC as the $y$ reserve, the prices at each of the ticks are:

$$\sqrt{p_c} = \sqrt{\frac{5000}{1}} = \sqrt{5000} \approx 70.71$$

$$\sqrt{p_l} = \sqrt{\frac{4500}{1}} \approx 67.08$$

$$\sqrt{p_u} = \sqrt{\frac{5500}{1}} \approx 74.16$$

Where $p_c$ is the current price, $p_l$ is the lower bound of the range, $p_u$ is the upper bound of the range.

Now, we can find corresponding ticks. We know that prices and ticks are connected via this formula:

$$\sqrt{p(i)}=1.0001^{\frac{i}{2}}$$

Thus, we can find tick $i$ via:

$$i = log_{\sqrt{1.0001}} \sqrt{p(i)}$$

> The square roots in this formula cancel out, but since we're working with $\sqrt{p}$ we need to preserve them.

Let's find the ticks:
1. Current tick: $i_c = log_{\sqrt{1.0001}} 70.71 = 85176$
1. Lower tick: $i_l = log_{\sqrt{1.0001}} 67.08 = 84122$
1. Upper tick: $i_u = log_{\sqrt{1.0001}} 74.16 = 86129$

> To calculate these, I used Python:
> ```python
>  import math
>  math.log(math.sqrt(5000), math.sqrt(1.0001))
>  > 85176.19043995449
>```
> Feel free using any other language.

That's it for price range calculation!

Last thing to note here is that Uniswap uses Q64.96 number to store $\sqrt{p}$. This is a fixed point number that has
64 bits for the integer part and 96 bits for the fractional part. In our above calculations, prices are floating point
numbers: `70.71`, `67.08`, `74.16`. We need to convert them to Q64.96. Luckily, this is simple: we need to multiply the
numbers by the maximum value of the fractional part of Q64.96, which is $2^{96}$. We'll get:

$$\sqrt{p_c} = 5602277097478614198912276234240$$

$$\sqrt{p_l} = 5314786713428871004159001755648$$

$$\sqrt{p_u} = 5875717789736564987741329162240$$

> In Python:
> ```python
> int(math.sqrt(5000) * 2**96)
> > 5602277097478614198912276234240
> ```
> Notice that we're multiplying before converting to integer. Otherwise, we'll use precision.

Ok, we're done here.

## Token Amounts Calculation

Next step is to decide how many tokens we want to deposit into the pool. The answer is: as many as we want. The amounts
are not strictly defined, we can deposit as much as it is enough to buy a small amount of ETH without causing the price
leave the price range we put liquidity into. During development and testing we'll be able to mint any amount of tokens,
so getting the amounts we want is not a problem.

For our first swap, let's deposit 1 ETH and 5000 USDC.

## Liquidity Amount Calculation

Next, we need to calculate $L$ based on the amounts we'll deposit.

From the theoretical introduction, you remember that:
$$L = \sqrt{xy}$$

However, we cannot simply multiply 1 ETH by 5000 USDC and take the square root. The reason is that the $x$ and $y$ in this
formula are **virtual reserves**.
[TODO: what are virtual reserves?]

We need to calculate $L$ specifically for the price range we're going to deposit liquidity into, and it'll be calculated
based on the amounts we're going to deposit. To find $L$, we need to look at one interesting fact: when the current price
equals the lower or the upper price, **one of the pool reserves is 0 and all pool's liquidity is in the other reserve**.
For example, if the current price is <span>$5500</span> then all ETH was bought from the pool and there's only USDC left.
And vice versa: when the current price is <span>$4500</span> then all USDC was bought from the pool and there's only ETH.

[TODO: illustrate]

[TODO: or maybe use the delta x and delta y formulas?]
$$\Delta x = \frac{L}{\sqrt{p(i_u)}} - \frac{L}{\sqrt{p(i_c)}} = \frac{L(\sqrt{p(i_u)} - \sqrt{p(i_c)})}{\sqrt{p(i_u)}\sqrt{p(i_c)}}$$
$$\Delta y = L\sqrt{p(i_c)} - L\sqrt{p(i_l)} = L(\sqrt{p(i_c)} - \sqrt{p(i_l)})$$

Knowing this, let's return to the trading formula of real reserves:

$$(x_{real} + \frac{L}{\sqrt{p_b}})(y_{real} + L\sqrt{p_a}) = L^{2}$$

So, there are two possible situations:
1. $x_{real}$ can be 0 when the entire reserve of $x$ is bought from the pool.
1. $y_{real}$ can be 0 when the entire reserve of $y$ is bought from the pool.

And these situations also serve as constraints: the amount of $L$ we deposit **must** satisfy both of them.

So, to find $L$, we need to calculate it in both of these scenarios. Let's begin with the one where $y$ is zero. The
trade function will look like so:

$$(x+\frac{L}{\sqrt{p_b}})L\sqrt{p_a} = L^{2}$$

When $y$ is zero, any trade will add some $\Delta y$ ($L\sqrt{p_a}$) to the empty reserve of $y$, and no buying of $y$ in
this situation is possible.

Next, we can find $L$:

$$L = x\frac{\sqrt{p_a}\sqrt{p_b}}{\sqrt{p_b}-\sqrt{p_a}}$$

Now, let's find a similar formula for the situation when $x$ is zero:

$$\frac{L}{\sqrt{p_b}}(y_{real} + L\sqrt{p_a}) = L^{2}$$
$$L = \frac{y_{real}}{\sqrt{p_b}-\sqrt{p_a}}$$

[TODO: show the calculations]

Having these two $L's$, we need to choose one of them and we'll choose the smaller one. Why? The amount of liquidity
we deposit must allow equally big price movements in both directions. If we pick the bigger amount, the other on won't
be enough to satisfy this requirement.

Now, let's plug our numbers into the formulas. For $x$, $p_a$ is the current price, and $p_b$ is the upper bound of the
price range. For $y$, $p_a$ is the lower bound and $p_b$ is the current price.

[TODO: add graph, x_real, y_real, from the whitepaper]

$$L = x\frac{\sqrt{p_a}\sqrt{p_b}}{\sqrt{p_b}-\sqrt{p_a}} = 1 ETH * \frac{67.08 * 70.71}{70.71 - 67.08}$$
After converting to Q64.96, we get:

$$L = 1519655488681761046528$$

Solving the other $L$:
$$L = \frac{y_{real}}{\sqrt{p_b}-\sqrt{p_a}} = \frac{5000USDC}{74.16-70.71}$$
$$L = 1377504647646213046272$$

> In Python:
> ```python
> def tick_to_sqrtp(i):
>     return int(1.0001**(i/2) * 2**96)
>
> pl = tick_to_sqrtp(84122)
> pc = tick_to_sqrtp(85176)
> pu = tick_to_sqrtp(86129)
> 
> q96 = 0x1000000000000000000000000
> liqX = ((1 * (10 ** 18)) * (pu * pc) / q96) / (pu - pc)
> liqY = (5000 * (10 ** 18)) * q96 / (pc - pl)
> 
> int(min(x,y))
> > 1377504647646213046272
> ```

Of these two we're picking the smaller one, `1377504647646213046272`.
