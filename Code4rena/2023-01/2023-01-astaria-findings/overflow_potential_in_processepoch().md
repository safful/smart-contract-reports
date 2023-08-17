## Tags

- bug
- 2 (Med Risk)
- satisfactory
- selected for report
- sponsor confirmed
- M-15

# [Overflow potential in processEpoch()](https://github.com/code-423n4/2023-01-astaria-findings/issues/362) 

# Lines of code

https://github.com/code-423n4/2023-01-astaria/blob/1bfc58b42109b839528ab1c21dc9803d663df898/src/PublicVault.sol#L321-L337


# Vulnerability details

## Impact
In `PublicVault.sol#processEpoch()`, we update the withdraw reserves based on how `totalAssets()` (the real amount of the underlying asset held by the vault) compares to the expected value in the withdraw proxy:

```
  unchecked {
    if (totalAssets() > expected) {
      s.withdrawReserve = (totalAssets() - expected)
        .mulWadDown(s.liquidationWithdrawRatio)
        .safeCastTo88();
    } else {
      s.withdrawReserve = 0;
    }
  }
```
In the event that the `totalAssets()` is greater than expected, we take the surplus in assets multiply it by the withdraw ratio, and assign this value to `s.withdrawReserve`.

However, because this logic is wrapped in an `unchecked` block, it must have confidence that this calculation does not overflow. Because the protocol allows arbitrary ERC20s to be used, it can't have confidence in the size of `totalAssets()`, which opens up the possibility for an overflow in this function.


## Proof of Concept

`mulWadDown` is calculated by first multiplying the two values, and then dividing by `1e18`. This is intended to prevent rounding errors with the division, but also means that an overflow is possible when the two values have been multiplied, before any division has taken place.

This unchecked block is safe from overflows if:
- `(totalAssets() - expected) * s.liquidationWithdrawRatio < 1e88` (because s.withdrawRatio is 88 bytes)
- `liquidationWithdrawRatio` is represented as a ratio of WAD, so it will be in the `1e17` - `1e18` range, let's assume `1e17` to be safe
- therefore we require `totalAssets() - expected < 1e61`

Although this is unlikely with most tokens (and certainly would have been safe in the previous iteration of the protocol that only used `WETH`, when allowing arbitrary ERC20 tokens, this is a risk.

## Tools Used

Manual Review

## Recommended Mitigation Steps

Remove the `unchecked` block around this calculation, or add an explicitly clause to handle the situation where `totalAssets()` gets too large for the current logic.