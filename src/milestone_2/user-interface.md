# User Interface

Let's make our web app work more like a real DEX. We can now remove hardcoded swap amounts and let users type arbitrary amounts. Moreover, we can now let users swap in both directions, so we also need a button to swap the token inputs.  After updating, the swap form will look like:

```jsx
<form className="SwapForm">
  <SwapInput
    amount={zeroForOne ? amount0 : amount1}
    disabled={!enabled || loading}
    readOnly={false}
    setAmount={setAmount_(zeroForOne ? setAmount0 : setAmount1, zeroForOne)}
    token={zeroForOne ? pair[0] : pair[1]} />
  <ChangeDirectionButton zeroForOne={zeroForOne} setZeroForOne={setZeroForOne} disabled={!enabled || loading} />
  <SwapInput
    amount={zeroForOne ? amount1 : amount0}
    disabled={!enabled || loading}
    readOnly={true}
    token={zeroForOne ? pair[1] : pair[0]} />
  <button className='swap' disabled={!enabled || loading} onClick={swap_}>Swap</button>
</form>
```

Each input has an amount assigned to it depending on the swap direction controlled by the `zeroForOne` state variable. The lower input field is always read-only because its value is calculated by the Quoter contract.

The `setAmount_` function does two things: it updates the value of the top input and calls the Quoter contract to calculate the value of the lower input:

```js
const updateAmountOut = debounce((amount) => {
  if (amount === 0 || amount === "0") {
    return;
  }

  setLoading(true);

  quoter.callStatic
    .quote({ pool: config.poolAddress, amountIn: ethers.utils.parseEther(amount), zeroForOne: zeroForOne })
    .then(({ amountOut }) => {
      zeroForOne ? setAmount1(ethers.utils.formatEther(amountOut)) : setAmount0(ethers.utils.formatEther(amountOut));
      setLoading(false);
    })
    .catch((err) => {
      zeroForOne ? setAmount1(0) : setAmount0(0);
      setLoading(false);
      console.error(err);
    })
})

const setAmount_ = (setAmountFn) => {
  return (amount) => {
    amount = amount || 0;
    setAmountFn(amount);
    updateAmountOut(amount)
  }
}
```

Notice the `callStatic` called on `quoter`–this is what we discussed in the previous chapter: we need to force Ethers.js to make a static call. Since `quote` is not a `pure` or `view` function, Ethers.js will try to call `quote` in a transaction.

And that's it! The UI now allows us to specify arbitrary amounts and swap in either direction!