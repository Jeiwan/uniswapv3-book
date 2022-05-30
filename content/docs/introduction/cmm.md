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
# Constant Function Market Makers

{{< katex display >}} {{</ katex >}}

As I mentioned in the previous section, there are different approach to building AMM. We'll be focusing and building one
specific type of AMM–Constant Function Market Maker. Don't be scared by the long name! At its core is a very
simply mathematical formula:

$$x * y = k$$

That's it, that's the formula.

\\(x\\) and \\(y\\) are pool contract reserves–the amount of tokens it current holds. *k* is just their product, actual
value doesn't matter.

> **Why there are only two reserves, *x* and *y*?**  
Each Uniswap pool can hold only two tokens. We use *x* and *y* to refer to reserves of one pool, where *x* is the reserve
of the first token and *y* is the reserve of the other token, and the order doesn't matter.

The constant function formula says: **after each trade, *k* must remain unchanged**. When traders make trades, they
put some amount of one token into pool (token they want to seel) and remove some amount of the other token from pool
(token they want to buy). This changes the reserves of the pool, and the constant function formula says that **the product**
of reserves must not change. As we will see many times in this book, this simple requirement makes the whole design work
and satisfies many non-obvious implications.

## The trade function
Now that we know what pools are, let's write a formula of how trading happens in a pool:

$$(x + r\Delta x)(y - \Delta y) = k\$$

[TODO: illustration]

1. There's a pool with some amount of token A (\\(x\\)) and some amount of token B (\\(y\\)).
1. When we buy token B for token A, we give some amount of token A to the pool (\\(\Delta x\\)).
1. The pool gives us some amount of token B in exchange (\\(\Delta y\\)).
1. The pool also takes a small fee from the amount of token A we gave (\\(r\\)).
1. The reserve of token A changes (\\(x + r \Delta x\\)), and the reserve of token B changes as well (\\(y - \Delta y\\)).
1. The product of updated reserves must still equal to \\(k\\).