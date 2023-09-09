## Tags

- bug
- 2 (Med Risk)
- primary issue
- selected for report
- sponsor confirmed
- M-04

# [Total assets of yearn vault are not correct](https://github.com/code-423n4/2023-01-popcorn-findings/issues/728) 

# Lines of code

https://github.com/code-423n4/2023-01-popcorn/blob/d95fc31449c260901811196d617366d6352258cd/src/vault/adapter/yearn/YearnAdapter.sol#L109-L111


# Vulnerability details

## Impact
Total assets of yearn vault are not correct so the calculation between the asset and the shares will be wrong.

## Proof of Concept

In `YearnAdapter` the total assets of current `yValut` are extracted using `_yTotalAssets`. 

```
    function _yTotalAssets() internal view returns (uint256) {
        return IERC20(asset()).balanceOf(address(yVault)) + yVault.totalDebt(); //@audit
    }
```

But in the yearn vault implementation, `self.totalIdle` is used instead of current balance.

```
def _totalAssets() -> uint256:
    # See note on `totalAssets()`.
    return self.totalIdle + self.totalDebt
```

In yearn valut, `totalIdle` is the tracked value of tokens, so it is same as vault's balance in most cases, but the balance can be larger due to an attack or other's fault. Even `sweep` is implemented for the case in the vault implementation. 

```
    if token == self.token.address:
        value = self.token.balanceOf(self) - self.totalIdle

    log Sweep(token, value)
    self.erc20_safe_transfer(token, self.governance, value)
```

So the result of `_yTotalAssets` can be inflated than the correct total assets and the calculation between the asset and the shares will be incorrect.


## Tools Used
Manual Review

## Recommended Mitigation Steps
Use `yVault.totalIdle` instead of balance.