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

The main difference between V2 and V3 is that, in V3, there are now **many curves**, not one. The curve we saw earlier
can now be seen as the curve of *virtual reserves*, and there are now many similar curves for *real reserves*. The full
price range (0 to infinity) is now split into many short ranges, each of which holds some amount of real reserves. And
each of these ranges **is bound by start and end prices** chosen by liquidity providers.

When looking at the curve, real reserves are amounts that bring the price (the price of X in terms of Y) to some points
$a$ and $b$:

[TODO: add curve with x_real, y_real]

If we sell many X tokens, the price will drop and reach the point $a$. If we buy many tokens X, the price will grow and
reach the point $b$. The range between $a$ and $b$ (including them) is one of the short ranges we can provide liquidity
into in Uniswap V3. We can graph this range as the original curve transitioned in such a way that it touches the axes:

[TODO: add virtual reserves -> real reserves transition graph]

Thus making it limited by the axes. It's no longer infinite, and **the reserves of an asset can be emptied now**. But
this is not a problem because this would happen only in one range and other ranges can still contain liquidity:

[TODO: add illustration, compare liquidity distributions]

This transitioned curve is the function:

$$(x + \frac{L}{\sqrt{p_b}})(y + L \sqrt{p_a}) = L^2$$

Don't confuse it with the trade function! This formula describes how real reserves are related to virtual ones. In this
formula, virtual reserves are shifted by $\frac{L}{\sqrt{p_b}}$ along the $x$ axis and by $L \sqrt{p_a}$ along the $y$
axis. What are these new variables?

$$L = \sqrt{xy}$$

$$\sqrt{P} = \sqrt{\frac{y}{x}}$$

Since token prices in a pool are reciprocals of each other, Uniswap V3 uses one definition of price: $\frac{y}{x}$, i.e.
the price of token X in terms of token Y. The price of token Y in terms of token X is simply $\frac{1}{y/x}=\frac{x}{y}$.
Similarly, $\frac{1}{\sqrt{P}} = \frac{1}{\sqrt{y/x}} = \sqrt{\frac{x}{y}}$. Thus, in the transition formula, the shifts
along axes are corresponding prices scaled by $L$, where $L$ is *a unit of liquidity*.

> In the previous chapter, we saw that the trade function can be rewritten as a comparison of geometric means of reserves
before and after a swap.

$L$ times price gives us the *amount of liquidity*. The curve of virtual reserves is shifted: along $x$ by the reserves of
token X; along $y$ by the reserves of token Y. The shifted curve crosses the axes in the coordinates equal to the amounts
of reserves, thus reserves can be depleted.

Why using $\sqrt{p}$ instead of $p$? There are two reasons:

1. Square root calculation is tricky and expensive in terms of gas consumption. Thus, it's easier to store the square root
without calculating it in the contracts.
1. $\sqrt{P}$ has an interesting connection to $L$: $L$ is also the relation between the change in output amount and 
the change in $\sqrt{P}$.

    $$L = \frac{\Delta y}{\Delta\sqrt{P}}$$

    [TODO: prove this]


## Pricing

The formula of the shifted curve tells us that real reserves are changes in $x$ and $y$ when transitioning from virtual
to real reserves. Thus, real reserves are:

$$x = \frac{L}{\sqrt{P}}$$
$$y = L \sqrt{P}$$

However, we'll never need to calculate them because $L$ and $\sqrt{P}$ allow us to find trade amounts without knowing
$x$ and $y$. Let's return to this formula:

$$L = \frac{\Delta y}{\Delta\sqrt{P}}$$

We can find $\Delta y$ from it:

$$\Delta y = \Delta \sqrt{P} L$$

As we discussed above, prices in a pool are reciprocals of each other. Thus, $\Delta x$ is:

$$\Delta x = \Delta \frac{1}{\sqrt{P}} L$$

$L$ and $\sqrt{P}$ allow us not to store and update pool reserves. Also, we don't need to calculate $\sqrt{P}$ each time
because we can always find $\Delta \sqrt{P}$ and its reciprocal.

## Ticks

Custom liquidity ranges are implemented through *ticks*–the range of all possible prices is demarcated by ticks and
liquidity is provided between two ticks chosen by a liquidity provider. 

[TODO: add illustration]

Each tick is mapped to some price from the range of all available prices. This mapping is done through this formula:

$$p(i) = 1.00001^i$$

Where $p(i)$ is the price at tick $i$. This formula is then improved to use $\sqrt{p}$ for the reasons explained above:

$$ \sqrt{p(i)} = \sqrt{1.0001}^i = 1.0001^{\frac{i}{2}}$$

$1.0001^i$ was chosen because it makes adjacent ticks 0.01% away from each other. **The difference in price between
two adjacent ticks is 0.01%**, or *1 basis point*.

> Basis point (1/100th of 1%, or 0.01%, or 0.0001) is a unit of measure of percentages in finance. You could heard about
basis point when central banks announced changes in interest rates.

This seemingly tricky algorithm allows to demarcate the continuous range of prices into evenly
distributed prices indexed by ticks, where the distance between two prices/ticks is always 0.01%. This is a powerful
design. If it's hard to understand now, it'll become clearer after we implemented it.