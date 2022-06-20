---
title: "Uniswap V3"
weight: 3
# bookFlatSection: false
# bookToc: true
# bookHidden: false
# bookCollapseSection: false
# bookComments: false
# bookSearchExclude: false
---
{{< katex display >}} {{</ katex >}}

# Introduction to Uniswap V3

> This chapter retells [the whitepaper of Uniswap V3](https://uniswap.org/whitepaper-v3.pdf). Again, it's totally ok if
you don't understand all the concepts. They will be clearer when converted to code.

To better understand the innovations Uniswap V3 brings, let's first look at the imperfections of Uniswap V2.

Uniswap V2 is a general exchange that implements one AMM algorithm. However, not all trading pairs are equal.
Pairs can be grouped by price volatility:

1. Tokens with medium and high price volatility. This group includes most tokens since most tokens don't have their
prices pegged to something and are subject to market fluctuations.
1. Tokens with low volatility. This group includes pegged tokens, mainly stablecoins: USDC-USDT, USDC-DAI, USDT-DAI, etc.
Also: ETH-stETH, ETH-rETH.

These groups require different, let's call them, pool configurations. The main difference is that pegged tokens require
high liquidity to reduce the demand effect (we learned about it in the previous chapter) on big trades. The prices of
USDC and USDT must stay close to 1, no matter how big the number of tokens we want to buy and sell. Since Uniswap V2's
general AMM algorithm is not very well suited for stablecoin trading, alternative AMMs (mainly [Curve](https://curve.fi))
were more popular for stablecoin trading.

What caused this problem is that liquidity in Uniswap V2 pools is distributed infinitely–pool liquidity allows trades at
any price, from 0 to infinity:

[TODO: add illustration]

This might not seem like a bad thing, but this reduces *capital efficiency*. Historical prices of an asset stay within
some defined range, whether it's narrow or wide. For example, the historical price range of ETH is from <span>$0.75</span>
to <span>$4,800</span> (according to [CoinMarketCap](https://coinmarketcap.com/currencies/ethereum/)). Today (June 2022,
 1 ETH costs <span>$1,1800</span>), no one would buy 1 ether at <span>$5000</span>, so it makes no sense to provide
liquidity at this price. Thus, a lot of liquidity in V2 pools cannot and won't be used, ever. And this can be improved.

## Concentrated Liquidity

Uniswap V3 introduces *concentrated liquidity*–liquidity providers can now choose the price range they want to provide
liquidity into. This improves capital efficiency by allowing to put more liquidity into a narrow price range, which makes
Uniswap more diverse: it can now have pools configured for pairs with different volatility. This fixes the problem of
Uniswap V2 we discussed above.

In a nutshell, Uniswap V3 is many small Uniswap V2s. The main difference between V2 and V3 is that, in V3, there are 
**many pools**, not one. Each of these smaller pools exists only within a certain *price range* and each of them has
**finite reserves**–we'll call them *real reserves*. The entire price range (from 0 to infinity) is can be filled with
these discrete pools, which provide liquidity within certain price ranges–this is the main feature of Uniswap V3.

[TODO: add illustration, compare liquidity distributions]

To set a price range, we need to pick two price points on the curve, $a$ and $b$:

[TODO: add curve with price a, b, and x_real, y_real]

As we saw in the previous chapter, buying or selling tokens moves the prices along the curve. A price range limits the
movement of the price. When the price moves to either of the points, the pool becomes depleted: one of the token reserves
will be 0 and buying this token won't be possible.

Let's look closely at the chart above:
1. The current price is the ratio of current reserves.
1. To get to point $a$ we need to buy all available $Y$; to get to point $b$ we need to buy all available $X$.
1. Real pool reserves are $x_real$ and $y_real$. These amounts define the current price and also allow to move the price
to one of the edge prices.

Since reserves can be depleted, this curve better illustrates the price ranges:

[TODO: add virtual reserves -> real reserves transition graph]

This is the original curve shifted in a way that makes it limited by the axes: the curve crosses the axes at the points
corresponding to the price range. This chart also better illustrated pool reserves: amounts required to move the price
of either of the tokens to one of the bounds of the price range.

What happens when the current price range gets depleted? The price slips into the next price range (if it exists, of
course). If the next price range doesn't exist, a trade is not possible. We'll see how this works later in the book.

To handle transitioning between price ranges, simplify liquidity management, and avoid rounding errors, Uniswap V3 uses
these new concepts:

$$L = \sqrt{xy}$$

$$\sqrt{P} = \sqrt{\frac{y}{x}}$$

$L$ can be seen as *the amount of liquidity*. In the previous chapter, we saw that the trade function can be rewritten as
a comparison of geometric means of reserves before and after a swap.

$\frac{y}{x}$ is the price of token X in terms of Y. Since token prices in a pool are reciprocals of each other,
we can use only one of them in calculations. The price of token Y in terms of token X is simply 
$\frac{1}{y/x}=\frac{x}{y}$. Similarly, $\frac{1}{\sqrt{P}} = \frac{1}{\sqrt{y/x}} = \sqrt{\frac{x}{y}}$.

$L$ times price gives us the *amount of liquidity*. The curve of virtual reserves is shifted: along $x$ by the reserves of
token X; along $y$ by the reserves of token Y. The real reserves curve crosses the axes in the coordinates equal to the
amounts of reserves, that's why reserves can be depleted.

Why using $\sqrt{p}$ instead of $p$? There are two reasons:

1. Square root calculation is not precise and causes rounding errors. Thus, it's easier to store the square root without
calculating it in the contracts.
1. $\sqrt{P}$ has an interesting connection to $L$: $L$ is also the relation between the change in output amount and 
the change in $\sqrt{P}$.

    $$L = \frac{\Delta y}{\Delta\sqrt{P}}$$

[TODO: prove this]

## Pricing

Pool reserves in Uniswap V3 are defined as:

$$x = \frac{L}{\sqrt{P}}$$
$$y = L \sqrt{P}$$

However, we'll never need to calculate them because $L$ and $\sqrt{P}$ allow us to find trade amounts without knowing
$x$ and $y$. Let's return to this formula:

$$L = \frac{\Delta y}{\Delta\sqrt{P}}$$

We can find $\Delta y$ from it:

$$\Delta y = \Delta \sqrt{P} L$$

As we discussed above, prices in a pool are reciprocals of each other. Thus, $\Delta x$ is:

$$\Delta x = \Delta \frac{1}{\sqrt{P}} L$$

$L$ and $\sqrt{P}$ allow us to not store and update pool reserves. Also, we don't need to calculate $\sqrt{P}$ each time
because we can always find $\Delta \sqrt{P}$ and its reciprocal.

## Ticks

Uniswap V3, however, doesn't allow us to select arbitrary prices when providing liquidity. Instead, it implements a scale
and we choose certain marks on it.

The entire price range is demarcated by evenly distributed discrete *ticks*. Each tick has an index and corresponds to
a certain price:

$$p(i) = 1.0001^i$$

Where $p(i)$ is the price at tick $i$. Taking powers of 1.0001 has a desirable property: the difference between two
adjacent ticks is 0.01% or *1 basis point*.

> Basis point (1/100th of 1%, or 0.01%, or 0.0001) is a unit of measure of percentages in finance. You could've heard about
basis point when central banks announced changes in interest rates.

As we discussed above, Uniswap V3 stores $\sqrt{P}$, not $P$. Thus, the formula is in fact:

$$\sqrt{p(i)} = \sqrt{1.0001}^i = 1.0001 ^{\frac{i}{2}}$$

So, we get values like: $\sqrt{p(0)} = 1$, $\sqrt{p(1)} = \sqrt{1.0001} \approx 1.00005$, $\sqrt{p(-1)} \approx 0.99995$.

Ticks are integers that can be positive and negative and, of course, they're not infinite. Ticks are mapped to prices,
thus they're limited by the price range. Uniswap V3 stores $\sqrt{P}$ as a fixed point Q64.96 number, which is a rational
number that uses 64 bits for the integer part and 96 bits for the fraction part. It's stored in an `uint160` variable and
it supports prices between $2^{-128}$ and $2^{128}$. Thus, the tick range is:

$$[log_{1.0001}2^{-128}, log_{1.0001}{2^{128}}] = [-887272, 887272]$$