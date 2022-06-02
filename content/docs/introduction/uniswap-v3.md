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

To better understand the innovations Uniswap V3 brings, let's first look at the imperfections of Uniswap V2.

Uniswap V2 is a general exchange that implements one AMM algorithm. However, not all trading pairs are equal. We can group
pairs by price volatility:

1. Pairs with medium and high volatility. This group includes most of the tokens since most tokens don't have their
prices pegged to something and are subject to market fluctuations.
1. Pairs with low volatility. This group includes pegged tokens, mainly stablecoins: USDC-USDT, USDC-DAI, USDT-DAI, etc.
Also: ETH-stETH, ETH-rETH.

These groups require different, let's call them, pool configurations. The main difference is that pegged tokens require
high liquidity to reduce the demand effect (we learned about it in the previous chapter) on big trades. The prices of
USDC and USDT must stay close to 1, no matter how big the number of tokens we want to buy and sell.

Another imperfection of Uniswap V2 is that liquidity in a pool is distributed infinitely. Liquidity is provided at any
price from 0 to infinity:

[TODO: add illustration]

[TODO: escape $]

This might not seem like a bad thing, but this reduces *capital efficiency*. Historical prices of an asset stay within
some defined range, whether it's narrow or wide. For example, the historical price range of ETH is from $0.75 to $4,800
(according to [CoinMarketCap](https://coinmarketcap.com/currencies/ethereum/)). Today (June 2022, 1 ETH costs $1,1800),
no one will buy 1 ether for $5000 so it makes no sense to provide liquidity at this price. Thus, a lot of liquidity in
a V2 pool cannot and won't be used, ever. And this can be improved.

## Concentrated Liquidity

Uniswap V3 introduces the idea of *concentrated liquidity*. Liquidity providers can now choose the price range they want
to provide liquidity into. This improves capital efficiency by allowing to put more liquidity into a narrow price range,
which makes Uniswap more diverse: it can now have pools configured specifically for medium-to-high volatility pairs or
low-volatility pairs. This fixes the problem of Uniswap V2 we discussed above. However, this also feature doesn't come
without drawbacks: it's now possible that the global price of an asset moves rapidly to a range V3 pools have no liquidity
in.

[TODO: add illustration, compare liquidity distributions]

Implementing concentrated liquidity requires changing the mathematics of Uniswap. Pool contracts no longer track pool
reserves, $x$ and $y$. Instead, they track liquidity, $L$:

$$L = \sqrt{xy}$$

And square root of price, $\sqrt{P}$:
$$\sqrt{P} = \sqrt{\frac{y}{x}}$$

$L$ is simply the geometric mean of reserves, you can think of it as a unit of liquidity.

> In the previous chapter, we figured out that the trade function can be rewritten as a comparison of geometric means of
reserves before and after a swap.

$P$ is the price of token X in terms of token Y, and $\sqrt{P}$ is the square root of this price. We don't need to track
both prices because they're simply reciprocals of each other: the price of token Y is
$\frac{1}{\sqrt{P}} = \frac{1}{\sqrt{y/x}}$.

[TODO: explain why using sqrt(P)]

These values can be used to compute the reserves:
$$x = \frac{L}{\sqrt{P}}$$
$$y = L \sqrt{P}$$

[TODO: explain calculations]

## Ticks

Custom liquidity ranges are implemented through *ticks*–the range of all possible prices is demarcated by ticks and
liquidity is provided between two ticks chosen by a liquidity provider. 

[TODO: add illustration]

Each tick is mapped to some price from the range of all available prices. This mapping is done through this formula:

$$p(i) = 1.00001^i$$

Where $p(i)$ is the price at tick $i$. This formula is then improved to use $\sqrt{p}$ instead of $p$ for two reasons:

1. Square root calculation is tricky and expensive in terms of gas consumption. Thus, it's easier to store the square root
of it.
1. $\sqrt{P}$ has an interesting connection to $L$:

    $$L = \frac{\Delta y}{\Delta\sqrt{P}}$$

    $L$ is also the relation between the change in output amount and $\sqrt{P}$.
    [TODO: prove this]

Thus, we can rewrite the tick-price formula as:

$$ \sqrt{p(i)} = \sqrt{1.0001}^i = 1.0001^{\frac{i}{2}}$$

$1.0001^i$ was chosen because it makes adjacent ticks 0.01% away from each other. **The difference in price between
two adjacent ticks is 0.01%**, or *1 basis point*.

> Basis point (1/100th of 1%, or 0.01%, or 0.0001) is a unit of measure of percentages in finance. You could heard about
basis point when central banks announced changes in interest rates.

The takeaway is: this seemingly tricky algorithm allows to demarcate the continuous range of prices into evenly
distributed prices indexed by ticks, where the distance between two prices/ticks is always 0.01%. This is a powerful
design. If it's hard to understand now, it'll become clearer after we implemented it.


## Pricing

The last thing we need to figure out is how prices (or output amounts) are calculated.

Since we can find $x$ and $y$ from $L$ and $\sqrt{P}$, we can calculate $\Delta x$ and $\Delta y$ using the formulas from
the previous chapter. However, there's a better way. Let's return to this formula:

$$L = \frac{\Delta y}{\Delta\sqrt{P}}$$

We can find $\Delta y$ using this formula:

$$\Delta y = \Delta \sqrt{P} L$$

As we discussed above, prices in a pool are reciprocals of each other. Since $\frac{y}{x}$ is the price of $x$ in terms
of $y$, then $\frac{x}{y}$ (or $\frac{1}{y/x}$) is the other way around. Next, if $\sqrt{P} = \sqrt{\frac{y}{x}}$, then
$\frac{1}{\sqrt{P}} = \frac{1}{\sqrt{y/x}}$, then $\frac{1}{\sqrt{P}}$ is the square root of the price of $y$ in terms
of $x$ (reciprocal of $\sqrt{P}$). Knowing this, we can express $L$ in terms of $\Delta x$:

$$L = \frac{\Delta x}{\Delta \frac{1}{\sqrt{P}}}$$

And, finally:

$$\Delta x = \Delta \frac{1}{\sqrt{P}} L$$

This means that we don't need to store reserves in the contract, and we also don't need to calculate the square root of P.
To calculate output amount we need to know input amount (trader chooses it) and $L$ (calculated based on liquidity in a pool).
To calculate input amount we need to know output amount (trader chooses it) and $L$. In both cases, we first need to
calculate $\Delta \sqrt{P}$ or $\Delta \frac{1}{\sqrt{P}}$.

[TODO: clean up, make the connection between the formulas clearer.]

## Summary
