---
title: "Constant Function Market Makers"
weight: 2
# bookFlatSection: false
# bookToc: true
# bookHidden: false
# bookCollapseSection: false
# bookComments: false
# bookSearchExclude: false
---
{{< katex display >}} {{</ katex >}}

# Constant Function Market Makers

> This chapter retells [the whitepaper of Uniswap V2](https://uniswap.org/whitepaper.pdf). Understanding this math is
crucial to build a Uniswap-like DEX, but it's totally fine if you don't understand everything at this stage.

As I mentioned in the previous section, there are different approaches to building AMM. We'll be focusing on and
building one specific type of AMM–Constant Function Market Maker. Don't be scared by the long name! At its core is a very
simple mathematical formula:

$$x * y = k$$

That's it, that's the formula.

$x$ and $y$ are pool contract reserves–the amounts of tokens it currently holds. *k* is just their product, actual
value doesn't matter.

> **Why there are only two reserves, *x* and *y*?**  
Each Uniswap pool can hold only two tokens. We use *x* and *y* to refer to reserves of one pool, where *x* is the reserve
of the first token and *y* is the reserve of the other token, and the order doesn't matter.

The constant function formula says: **after each trade, *k* must remain unchanged**. When traders make trades, they
put some amount of one token into a pool (token they want to sell) and remove some amount of the other token from the pool
(token they want to buy). This changes the reserves of the pool, and the constant function formula says that **the product**
of reserves must not change. As we will see many times in this book, this simple requirement is the core algorithm of the
exchange we're building.

## The trade function
Now that we know what pools are, let's write the formula of how trading happens in a pool:

$$(x + r\Delta x)(y - \Delta y) = k\$$

[TODO: illustration]

1. There's a pool with some amount of token A ($x$) and some amount of token B ($y$).
1. When we buy token B for token A, we give some amount of token A to the pool ($\Delta x$).
1. The pool gives us some amount of token B in exchange ($\Delta y$).
1. The pool also takes a small fee from the amount of token A we gave ($r$).
1. The reserve of token A changes ($x + r \Delta x$), and the reserve of token B changes as well ($y - \Delta y$).
1. The product of updated reserves must still equal $k$.

The order of tokens in the formula doesn't matter: Uniswap pools allow swapping tokens in both directions.

## Pricing

How do we calculate the prices of tokens in a pool?

Since Uniswap pools are separate smart contracts, **tokens in a pool are priced in terms of each other**. For example: in
USDC-ETH pool, ETH is priced in terms of USDC and USDC is priced in terms of ETH. If 1 ETH costs 1000 USDC, then 1 USDC
costs 0.001 ETH. The same is true for any other pool, whether it's a stablecoin pair or not. And actual token prices
are simply relations of reserves:

$$P_x = \frac{y}{x}, \quad P_y=\frac{x}{y}$$

Where $P_x$ and $P_y$ are prices of tokens in terms of the other token. Such prices are called *spot prices* and they
only reflect current market prices. However, the actual price of a trade is calculated differently. Let's return to the
trade function and try to come up with some conclusions about how an actual trade price is calculated:

$$(x + r\Delta x)(y - \Delta y) = k\$$

Suppose we want to find the price of token A (its reserve is $x$ in the formula) when swapping it for token B (its
reserve is $y$ in the formula). We're trading in some amount of token A ($\Delta x$) in exchange for some amount of
token B ($\Delta y$). This means that the actual price of the trade will be **the relation of the amounts**. Not the
reserves, but the amounts we give and get.

Let's rewrite the trade function to find out trade amounts:
1. First, we write $k$ as the product of reserves before a trade:
    $$(x + r\Delta x)(y - \Delta y) = xy\$$
    On the left side, is the product of updated reserves (after a swap). On the right side, is the product of current
    reserves (before a swap).
1. Then, we can find $\Delta y$ using simple algebraic operations:
    $$\Delta y = \frac{yr\Delta x}{x + r\Delta x}$$
    [TODO: explain]
1. Similarly, we can express $\Delta x$ in terms of $\Delta y$:
    $$\Delta x = \frac{x \Delta y}{r(y - \Delta y)}$$
    [TODO: explain]

Having these functions, we don't need to calculate prices because we can calculate amounts instead! If we know how many
tokens we want to sell, we can calculate the amount we'll get without calculating the price. And vice versa: if we want
to buy a specific amount of tokens, we can calculate the amount we need to sell right away, without calculating the price.

Last thing to notice here is that the trade function can be rewritten using geometric means. So this formula:
$$(x + r\Delta x)(y - \Delta y) = xy\$$

Becomes this:
$$\sqrt[n]{\prod_{i=1}^n X_i'} = \sqrt[n]{\prod_{i=1}^n X_i}$$

Where: $n=2$ (since we have only two tokens in a pool), $X_i$ is current reserves ($X_1 = x, X_2=y$), $X_i'$ is updated
reserves. This is a general representation of the trade function, Uniswap's implementation is a special case of this
formula.


## The Curve

The above calculations might seem too abstract and dry. Let's visualize the constant product function to better understand
how it works

When plotted, the constant product function is a quadratic hyperbola:

[TODO: add graph]

Where axes are reserves. Every trade starts at the point on the curve that corresponds to the current ratio of reserves.
To calculate the output amount, we need to find a new point on the curve, which has the $x$ coordinate of $x+\Delta x$, i.e.
current reserve of token A + the amount we're selling. The change in $y$ is the number of tokens B we'll get.

Let's look at a concrete example:

[TODO: add graph]

1. Start price ($P_x = \frac{y}{x}$) is 4: 1 X = 4 Y.
1. We're selling 42 X. If we use only the start price, we expect to get 42 * 4 = 168 Y.
1. However, the execution price is 2.173, so we get only 91.304 Y!

> To build a better intuition of how it works, try making up several scenario and plot them on the graph. Try different
X amount relative to the reserve of X, see how output amount changes hen $\Delta x$ is small relative to $x$.

> This wonderful chart was created by [Dan Robinson](https://twitter.com/danrobinson), one of the creators of Uniswap. 
Massive kudos!

I bet you're wondering why using such a curve? It might seem like it punishes you for trading big amounts. This is true,
and this is a desirable property! The law of supply and demand tells us that when demand is high (and supply is constant)
the price is also high. And when demand is low, the price is also lower. This is how markets work. And, magically,
the constant product function implements this mechanism! Demand is defined by the amount you want to buy, and supply is the
pool reserves. When you want to buy a big amount relative to pool reserves the price is higher than when you want to
buy a smaller amount. Such a simple formula guarantees such a powerful mechanism!

**And that's the whole math of Uniswap! Phew!**

Well, this is the math of Uniswap V2, and we're studying Uniswap V3. So in the next part, we'll see how the mathematics
of Uniswap V3 is different.