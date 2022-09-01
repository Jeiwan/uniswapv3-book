---
title: "Protocol Fees"
weight: 4
# bookFlatSection: false
# bookToc: true
# bookHidden: false
# bookCollapseSection: false
# bookComments: false
# bookSearchExclude: false
---

{{< katex display >}} {{</ katex >}}

# Protocol Fees

While working on the Uniswap implementation, you've probably asked yourself, "How does Uniswap make money?" Well, it
doesn't (at least as of September 2022).

In the implementation we've built so far, traders pay liquidity providers for providing liquidity, and Uniswap Labs, as
the company that developed the DEX, is not part of this process. Neither traders, nor liquidity providers pay Uniswap
Labs for using the Uniswap DEX. How come?

In fact, there's a way for Uniswap Labs to start making money on the DEX. However, the mechanism hasn't been enabled yet
(again, as of September 2022). Each Uniswap pool has *protocol fees* collection mechanism. Protocol fees are collected
from swap fees: a small portion of swap fees is subtracted and saved as protocol fees to later be collected by Factory
contract owner (Uniswap Labs). The size of protocol fees is expected to be determined by UNI token holders, but it must
be between $1/4$ and $1/10$ (inclusive).

For brevity, we're not going to add protocol fees to our implementation, but let's see how they're implemented in Uniswap.

Protocol fee size is stored in `slot0`:

```solidity
// UniswapV3Pool.sol
struct Slot0 {
    ...
    // the current protocol fee as a percentage of the swap fee taken on withdrawal
    // represented as an integer denominator (1/x)%
    uint8 feeProtocol;
    ...
}
```

And we'll also need a global accumulator to track accrued fees:
```solidity
// accumulated protocol fees in token0/token1 units
struct ProtocolFees {
    uint128 token0;
    uint128 token1;
}
ProtocolFees public override protocolFees;
```

Protocol fees can be set in `setFeeProtocol` function:

```solidity
function setFeeProtocol(uint8 feeProtocol0, uint8 feeProtocol1) external override lock onlyFactoryOwner {
    require(
        (feeProtocol0 == 0 || (feeProtocol0 >= 4 && feeProtocol0 <= 10)) &&
            (feeProtocol1 == 0 || (feeProtocol1 >= 4 && feeProtocol1 <= 10))
    );
    uint8 feeProtocolOld = slot0.feeProtocol;
    slot0.feeProtocol = feeProtocol0 + (feeProtocol1 << 4);
    emit SetFeeProtocol(feeProtocolOld % 16, feeProtocolOld >> 4, feeProtocol0, feeProtocol1);
}
```

As you can see, it's allowed to set protocol fees separate for each of the tokens. The values are two `uint8` that are
packed to be stored in one `uint8`: `feeProtocol1` is shifted to the left by 4 bits (this is identical to multiplying it
by 16) and added to `feeProtocol0`. To unpack `feeProtocol0`, a remainder of division `slot0.feeProtocol` by 16 is taken;
`feeProtocol1` is simply integer division of `slot0.feeProtocol`  by 4.

Before beginning a swap, we need to choose one of the protocol fees depending on swap direction (swap and protocol fees
are collected on input tokens):

```solidity
function swap(...) {
    ...
    uint8 feeProtocol = zeroForOne ? (slot0_.feeProtocol % 16) : (slot0_.feeProtocol >> 4);
    ...
```

To accrue protocol fees, we subtract them from swap fees right after computing swap step amounts:

```solidity
...
while (...) {
    (..., step.feeAmount) = SwapMath.computeSwapStep(...);

    if (cache.feeProtocol > 0) {
        uint256 delta = step.feeAmount / cache.feeProtocol;
        step.feeAmount -= delta;
        state.protocolFee += uint128(delta);
    }

    ...
}
...
```

After a swap is done, the protocol fees accumulator needs to be updated:
```solidity
if (zeroForOne) {
    if (state.protocolFee > 0) protocolFees.token0 += state.protocolFee;
} else {
    if (state.protocolFee > 0) protocolFees.token1 += state.protocolFee;
}
```

Finally, Factory contract owner can collect accrued protocol fees:

```solidity
function collectProtocol(
    address recipient,
    uint128 amount0Requested,
    uint128 amount1Requested
) external override lock onlyFactoryOwner returns (uint128 amount0, uint128 amount1) {
    amount0 = amount0Requested > protocolFees.token0 ? protocolFees.token0 : amount0Requested;
    amount1 = amount1Requested > protocolFees.token1 ? protocolFees.token1 : amount1Requested;

    if (amount0 > 0) {
        if (amount0 == protocolFees.token0) amount0--;
        protocolFees.token0 -= amount0;
        TransferHelper.safeTransfer(token0, recipient, amount0);
    }
    if (amount1 > 0) {
        if (amount1 == protocolFees.token1) amount1--;
        protocolFees.token1 -= amount1;
        TransferHelper.safeTransfer(token1, recipient, amount1);
    }

    emit CollectProtocol(msg.sender, recipient, amount0, amount1);
}
```