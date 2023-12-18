# Flash Loan Fees

In a previous chapter, we implemented flash loans and made them free. However, Uniswap collects swap fees on flash loans, and we're going to add this to our implementation: the amounts repaid by flash loan borrowers must include a fee.

Here's what the updated `flash` function looks like:
```solidity
function flash(
    uint256 amount0,
    uint256 amount1,
    bytes calldata data
) public {
    uint256 fee0 = Math.mulDivRoundingUp(amount0, fee, 1e6);
    uint256 fee1 = Math.mulDivRoundingUp(amount1, fee, 1e6);

    uint256 balance0Before = IERC20(token0).balanceOf(address(this));
    uint256 balance1Before = IERC20(token1).balanceOf(address(this));

    if (amount0 > 0) IERC20(token0).transfer(msg.sender, amount0);
    if (amount1 > 0) IERC20(token1).transfer(msg.sender, amount1);

    IUniswapV3FlashCallback(msg.sender).uniswapV3FlashCallback(
        fee0,
        fee1,
        data
    );

    if (IERC20(token0).balanceOf(address(this)) < balance0Before + fee0)
        revert FlashLoanNotPaid();
    if (IERC20(token1).balanceOf(address(this)) < balance1Before + fee1)
        revert FlashLoanNotPaid();

    emit Flash(msg.sender, amount0, amount1);
}
```

What's changed is that we're now calculating fees on the amounts requested by the caller and then expect pool balances to have grown by the fee amounts.