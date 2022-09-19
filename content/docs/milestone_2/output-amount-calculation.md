---
title: "Output Amount Calculation"
weight: 2
# bookFlatSection: false
# bookToc: true
# bookHidden: false
# bookCollapseSection: false
# bookComments: false
# bookSearchExclude: false
---

{{< katex display >}} {{</ katex >}}

# Output Amount Calculation

Our collection of Uniswap math formulas lacks a final piece: the formula of calculating the output amount when selling
ETH (that is: selling token $x$). In the previous milestone, we had an analogous formula for the scenario when ETH is
bought (buying token $x$):

$$\Delta \sqrt{P} = \frac{\Delta y}{L}$$

This formula finds the change in the price when selling token $y$. We then added this change to the current price to
find the target price:

$$\sqrt{P_{target}} = \sqrt{P_{current}} + \Delta \sqrt{P}$$

Now, we need a similar formula to find the target price when selling token $x$ (ETH in our case) and buying token $y$
(USDC in our case).

Recall that the change in token $x$ can be calculated as:

$$\Delta x = \Delta \frac{1}{\sqrt{P}}L$$

From this formula, we can find the target price:

$$\Delta x = (\frac{1}{\sqrt{P_{target}}} - \frac{1}{\sqrt{P_{current}}}) L$$
$$= \frac{L}{\sqrt{P_{target}}} - \frac{L}{\sqrt{P_{current}}}$$

From this, we can find $\sqrt{P_{target}}$ using basic algebraic transformations:

$$\sqrt{P_{target}} = \frac{\sqrt{P}L}{\Delta x \sqrt{P} + L}$$

Knowing the target price, we can find the output amount similarly to how we found it in the previous milestone.

Let's update our Python script with the new formula:
```python
# Swap ETH for USDC
amount_in = 0.01337 * eth

print(f"\nSelling {amount_in/eth} ETH")

price_next = int((liq * q96 * sqrtp_cur) // (liq * q96 + amount_in * sqrtp_cur))

print("New price:", (price_next / q96) ** 2)
print("New sqrtP:", price_next)
print("New tick:", price_to_tick((price_next / q96) ** 2))

amount_in = calc_amount0(liq, price_next, sqrtp_cur)
amount_out = calc_amount1(liq, price_next, sqrtp_cur)

print("ETH in:", amount_in / eth)
print("USDC out:", amount_out / eth)
```

Its output:
```shell
Selling 0.01337 ETH
New price: 4993.777388290041
New sqrtP: 5598789932670289186088059666432
New tick: 85163
ETH in: 0.013369999999998142
USDC out: 66.80838889019013
```

Which means that we'll get 66.8 USDC when selling 0.01337 ETH using the liquidity
we provided in the previous step.

This looks good, but enough of Python! We're going to implement all the
math calculations in Solidity.