## Tags

- bug
- 2 (Med Risk)
- high quality report
- primary issue
- selected for report
- sponsor confirmed
- M-06

# [_computeAvailable() the calculations are wrong](https://github.com/code-423n4/2023-08-pooltogether-findings/issues/90) 

# Lines of code

https://github.com/GenerationSoftware/pt-v5-vault-boost/blob/9d640051ab61a0fdbcc9500814b7f8242db9aec2/src/VaultBooster.sol#L277


# Vulnerability details

## Impact
`_computeAvailable()` incorrect calculations that result in a return value greater than the current balance, causing methods such as `liquidate` to fail

## Proof of Concept
`VaultBooster._computeAvailable()` used to count the number of `tokens` currently available
There are two conditions

1. `accrue` continuously according to time
2. the final cumulative value cannot be greater than the current contract balance

The code is as follows:
```solidity
  function _computeAvailable(IERC20 _tokenOut) internal view returns (uint256) {
    Boost memory boost = _boosts[_tokenOut];
    uint256 deltaTime = block.timestamp - boost.lastAccruedAt;
    uint256 deltaAmount;
    if (deltaTime == 0) {
      return boost.available;
    }
    if (boost.tokensPerSecond > 0) {
      deltaAmount = boost.tokensPerSecond * deltaTime;
    }
    if (boost.multiplierOfTotalSupplyPerSecond.unwrap() > 0) {
      uint256 totalSupply = twabController.getTotalSupplyTwabBetween(address(vault), uint32(boost.lastAccruedAt), uint32(block.timestamp));
      deltaAmount += convert(boost.multiplierOfTotalSupplyPerSecond.intoUD60x18().mul(convert(deltaTime)).mul(convert(totalSupply)));
    }
@>  uint256 availableBalance = _tokenOut.balanceOf(address(this));
@>  deltaAmount = availableBalance > deltaAmount ? deltaAmount : availableBalance;
    return boost.available + deltaAmount;
  }
```

The current implementation code, limiting the maximum value of `deltaAmount` is wrong, using the minimum value compared to the current balance `_tokenOut.balanceOf(address(this))`.
But the current balance includes the previously accumulated `boost.available`, so normally it should be compared to the difference between the current balance and `boost.available`.

so the value returned may be larger than the current balance, and `LiquidationPair.sol ` performs `source.liquidatableBalanceOf()` and `source.liquidate()` with too large a number, resulting in a failed transfer.

## Tools Used

## Recommended Mitigation Steps

The maximum value returned should not exceed the current balance


```solidity
  function _computeAvailable(IERC20 _tokenOut) internal view returns (uint256) {
    Boost memory boost = _boosts[_tokenOut];
    uint256 deltaTime = block.timestamp - boost.lastAccruedAt;
    uint256 deltaAmount;
    if (deltaTime == 0) {
      return boost.available;
    }
    if (boost.tokensPerSecond > 0) {
      deltaAmount = boost.tokensPerSecond * deltaTime;
    }
    if (boost.multiplierOfTotalSupplyPerSecond.unwrap() > 0) {
      uint256 totalSupply = twabController.getTotalSupplyTwabBetween(address(vault), uint32(boost.lastAccruedAt), uint32(block.timestamp));
      deltaAmount += convert(boost.multiplierOfTotalSupplyPerSecond.intoUD60x18().mul(convert(deltaTime)).mul(convert(totalSupply)));
    }
    uint256 availableBalance = _tokenOut.balanceOf(address(this));
-   deltaAmount = availableBalance > deltaAmount ? deltaAmount : availableBalance;
-   return boost.available + deltaAmount;
+   uint256 result = boost.available + deltaAmount;
+   if (result > availableBalance) result = availableBalance;
+   return result;

  }
```


## Assessed type

Context