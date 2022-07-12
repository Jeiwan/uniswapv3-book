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

{{< katex display >}} {{</ katex >}}

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

### Finding Next Tick

Next step is finding ticks with liquidity using the bitmap index.

During swapping, we need to find a tick with liquidity that's before or after the current tick (that is: to the left or
to the right of it). In the previous milestone, we used to [calculate and hard code it](https://github.com/Jeiwan/uniswapv3-code/blob/85b8605c37a9065c141a234ee2c18d9507eeba22/src/UniswapV3Pool.sol#L142),
but now we need to find such tick using the bitmap index. We'll do this in `TickMath.nextInitializedTickWithinOneWord`
method. In this function, we'll need to implement two scenarios:

1. When selling token $X$ (ETH in our case), find next initialized tick in the current tick's word and **to the right** of the current tick.
1. When selling token $Y$ (USDC in our case), find next initialized tick in the next (current + 1) tick's word and **to the left** of the current tick.

This corresponds to the price movement when making swaps in either directions:

[TODO: add illustration, price change direction and tick change direction]

We include the current price when price grows.

Now, let's look at the implementation:
```solidity
function nextInitializedTickWithinOneWord(
    mapping(int16 => uint256) storage self,
    int24 tick,
    int24 tickSpacing,
    bool lte
) internal view returns (int24 next, bool initialized) {
    int24 compressed = tick / tickSpacing;
    ...
```

1. First arguments makes this function a method of `mapping(int16 => uint256)`.
1. `tick` is the current tick.
1. `tickSpacing` is always 1 until we start using it in Milestone 3.
1. `lte` is the flag that sets the direction. When `true`, we're selling token $X$ and searching for next initialized tick
to the right of the current one. When `false,` it's the other way around.

```solidity
if (lte) {
    (int16 wordPos, uint8 bitPos) = position(compressed);
    uint256 mask = (1 << bitPos) - 1 + (1 << bitPos);
    uint256 masked = self[wordPos] & mask;
    ...
```

When selling $X$, we're:
1. taking current tick's word and bit positions;
1. making a mask where all bits to the right of the current bit position, including it, are ones (`mask` is all ones,
its length = `bitPos`);
1. applying the mask to the current tick's word.

```solidity
    ...
    initialized = masked != 0;
    next = initialized
        ? (compressed - int24(uint24(bitPos - BitMath.mostSignificantBit(masked)))) * tickSpacing
        : (compressed - int24(uint24(bitPos))) * tickSpacing;
    ...
```
Next, `masked` won't equal 0 if at least one bit of it is set to 1. If so, there's an initialized tick; if not, there
isn't (not in the current word). Depending on the result, we either return the index of the next initialized tick or the
leftmost bit in the next word–this will allow to search for initialized ticks in the word during another loop cycle.

```solidity
    ...
} else {
    (int16 wordPos, uint8 bitPos) = position(compressed + 1);
    uint256 mask = ~((1 << bitPos) - 1);
    uint256 masked = self[wordPos] & mask;
    ...
```

Similarly, when selling $Y$, we're:
1. taking next tick's word and bit positions;
1. making a different mask, where all bits to the left of next tick bit position are ones and all the bits to the right
are zeros;
1. applying the mask to the next tick's word.

Again, if there's no initialized ticks to the left, the rightmost bit of the previous word is returned:
```solidity
    ...
    initialized = masked != 0;
    // overflow/underflow is possible, but prevented externally by limiting both tickSpacing and tick
    next = initialized
        ? (compressed + 1 + int24(uint24((BitMath.leastSignificantBit(masked) - bitPos)))) * tickSpacing
        : (compressed + 1 + int24(uint24((type(uint8).max - bitPos)))) * tickSpacing;
}
```

And that's it!

As you can see, `nextInitializedTickWithinOneWord` doesn't find the exact tick–it's scope of search is current or next
tick's word. Indeed, we don't want to iterate over all the words since the we don't send boundaries on the bitmap index.
This function, however, plays well with `swap` function–soon, we'll see this.