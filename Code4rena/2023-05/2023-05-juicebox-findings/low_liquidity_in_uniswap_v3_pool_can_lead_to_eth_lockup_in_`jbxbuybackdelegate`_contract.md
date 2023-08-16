## Tags

- bug
- 2 (Med Risk)
- downgraded by judge
- primary issue
- selected for report
- sponsor confirmed
- M-02

# [Low Liquidity in Uniswap V3 Pool Can Lead to ETH Lockup in `JBXBuybackDelegate` Contract](https://github.com/code-423n4/2023-05-juicebox-findings/issues/162) 

# Lines of code

https://github.com/code-423n4/2023-05-juicebox/blob/9a36e5c8d0588f0f262a0cd1c08e34b2184d8f4d/juice-buyback/contracts/JBXBuybackDelegate.sol#L216


# Vulnerability details

## Impact

The `JBXBuybackDelegate` contract employs Uniswap V3 to perform ETH-to-project token swaps. When the terminal invokes the `JBXBuybackDelegate.didPay()` function, it provides the amount of ETH to be swapped for project tokens. The swap operation sets `sqrtPriceLimitX96` to the lowest possible price, and the slippage is checked at the callback.

However, if the Uniswap V3 pool lacks sufficient liquidity or being manipulated before the transaction is executed, the swap will halt once the pool's price reaches the `sqrtPriceLimitX96` value. Consequently, not all the ETH sent to the contract will be utilized, resulting in the remaining ETH becoming permanently locked within the contract.

## Proof of Concept

The `_swap()` function interacts with the Uniswap V3 pool. It sets `sqrtPriceLimitX96` to the minimum or maximum feasible value to ensure that the swap attempts to utilize all available liquidity in the pool.

```solidity
try pool.swap({
    recipient: address(this),
    zeroForOne: !_projectTokenIsZero,
    amountSpecified: int256(_data.amount.value),
    sqrtPriceLimitX96: _projectTokenIsZero ? TickMath.MAX_SQRT_RATIO - 1 : TickMath.MIN_SQRT_RATIO + 1,
    data: abi.encode(_minimumReceivedFromSwap)
}) returns (int256 amount0, int256 amount1) {
    // Swap succeeded, take note of the amount of projectToken received (negative as it is an exact input)
    _amountReceived = uint256(-(_projectTokenIsZero ? amount0 : amount1));
} catch {
    // implies _amountReceived = 0 -> will later mint when back in didPay
    return _amountReceived;
}
```

In the Uniswap V3 pool, [this check](https://github.com/Uniswap/v3-core/blob/main/contracts/UniswapV3Pool.sol#LL640C9-L650C15) stops the loop if the price limit is reached or the entire input has been used. If the pool does not have enough liquidity, it will still do the swap until the price reaches the minimum/maximum price.

```solidity
// continue swapping as long as we haven't used the entire input/output and haven't reached the price limit
while (state.amountSpecifiedRemaining != 0 && state.sqrtPriceX96 != sqrtPriceLimitX96) {
    StepComputations memory step;

    step.sqrtPriceStartX96 = state.sqrtPriceX96;

    (step.tickNext, step.initialized) = tickBitmap.nextInitializedTickWithinOneWord(
        state.tick,
        tickSpacing,
        zeroForOne
    );
```

Finally, the `uniswapV3SwapCallback()` function uses the input from the pool callback to wrap ETH and transfer WETH to the pool. So, if `_amountToSend < msg.value`, the unused ETH is locked in the contract.

```solidity
function uniswapV3SwapCallback(int256 amount0Delta, int256 amount1Delta, bytes calldata data) external override {
    // Check if this is really a callback
    if (msg.sender != address(pool)) revert JuiceBuyback_Unauthorized();

    // Unpack the data
    (uint256 _minimumAmountReceived) = abi.decode(data, (uint256));

    // Assign 0 and 1 accordingly
    uint256 _amountReceived = uint256(-(_projectTokenIsZero ? amount0Delta : amount1Delta));
    uint256 _amountToSend = uint256(_projectTokenIsZero ? amount1Delta : amount0Delta);

    // Revert if slippage is too high
    if (_amountReceived < _minimumAmountReceived) revert JuiceBuyback_MaximumSlippage();

    // Wrap and transfer the weth to the pool
    weth.deposit{value: _amountToSend}();
    weth.transfer(address(pool), _amountToSend);
}
```

## Tools Used

Manual Review

## Recommended Mitigation Steps

Consider returning the amount of unused ETH to the beneficiary.




## Assessed type

Other