---
title: "Tick Bitmap Index"
weight: 4
# bookFlatSection: false
# bookToc: true
# bookHidden: false
# bookCollapseSection: false
# bookComments: false
# bookSearchExclude: false
---

# Tick Bitmap Index

As the first step towards dynamic swaps, we need to implement an index of ticks. In the previous milestone, we used to
calculate the target tick when making a swap:
```solidity
function swap(address recipient, bytes calldata data)
    public
    returns (int256 amount0, int256 amount1)
{
  int24 nextTick = 85184;
  ...
}
```

When there's liquidity provided in different price ranges, we cannot simply calculate the target tick. We need to **find
it** depending on the amount of liquidity in different price ranges. Thus, we need to index all ticks that have liquidity
and then use the index to find ticks to "fill" enough liquidity for a swap. In this step, we're going to implement such
index.

## Bitmap

Bitmap is a popular technique of indexing data in a compact way. A bitmap is simply an array of zeros and ones, where
each element as and index and corresponds to some external entity (something that's indexed). Each element can be a zero
or a one, which can be seemed as setting a flag: when 0, flag is not set; when 1, flag is set. What makes this approach
favorable is that the whole array can be stored as a single number in the binary number system!

For example, the array `111101001101001` is number 31337. The number takes two bytes (0x7a69) and two bytes can store 16
flags (1 byte = 8 bits).

Uniswap V3 uses this technique to store the information about initialized ticks, that is ticks with some liquidity. When
a flag is set (1), the tick has liquidity; when flag is not set (0), the tick is not initialized. Let's review the
implementation.

## TickBitmap Contract

In the pool contract, the tick index is stored in a state variable:
```solidity
contract UniswapV3Pool {
    using TickBitmap for mapping(int16 => uint256);
    mapping(int16 => uint256) public tickBitmap;
    ...
}
```

This is mapping where keys are `int16`'s and values are words (`uint256`). Imagine an infinite continuous array of ones
and zeros:

[TODO: add illustration]

Each element in this array corresponds to a tick. To navigate in this array, we break it into words: sub-arrays of
length 256 bits. To find tick's position in this array, we do:

```solidity
function position(int24 tick) private pure returns (int16 wordPos, uint8 bitPos) {
    wordPos = int16(tick >> 8);
    bitPos = uint8(uint24(tick % 256));
}
```

That is: we find its word position and then its bit in this word. `>> 8` is identical to division by 256. So, word
position is the integer part of tick index divided by 256, and bit position is the remainder.

As an example, let's calculate word and bit positions for one of our ticks:
```python
tick = 85176
word_pos = tick >> 8 # or tick // 2**8
bit_pos = tick % 256
print(f"Word {word_pos}, bit {bit_pos}")
# Word 332, bit 184
```

### Flipping Flags

When adding liquidity into a pool, we need to set a couple of tick flags in the bitmap: one for the lower tick and one
for the upper tick. We do this in `flipTick` method of the bitmap mapping:
```solidity
function flipTick(
    mapping(int16 => uint256) storage self,
    int24 tick,
    int24 tickSpacing
) internal {
    require(tick % tickSpacing == 0); // ensure that the tick is spaced
    (int16 wordPos, uint8 bitPos) = position(tick / tickSpacing);
    uint256 mask = 1 << bitPos;
    self[wordPos] ^= mask;
}
```

> Until later in the book, `tickSpacing` is always 1.

After finding word and bit positions, we need to make a mask. A mask is a number that has a single 1 flag set at the
bit position of the tick. To find the mask, we simply calculate `2**bit_pos` (equivalent of `1 << bit_pos`):
```python
mask = 2**bit_pos # or 1 << bit_pos
print(bin(mask))
#0b10000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000
```

Next, to flip a flag, we apply the mask to the tick's word via bitwise XOR:
```python
word = (2**256) - 1 # set word to all ones
print(bin(word ^ mask))
#0b11111111111111111111111111111111111111111111111111111111111111111111111->0<-1111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111
```

You'll see that 184th bit (counting from the right starting at 0) has flipped to 0.