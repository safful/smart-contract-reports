## Tags

- bug
- 2 (Med Risk)
- disagree with severity
- downgraded by judge
- judge review requested
- primary issue
- selected for report
- sponsor confirmed
- edited-by-warden
- M-05

# [Virgin stake can claim all drops](https://github.com/code-423n4/2023-05-xeth-findings/issues/23) 

# Lines of code

https://github.com/code-423n4/2023-05-xeth/blob/d86fe0a9959c2b43c62716240d981ae95224e49e/src/wxETH.sol#L222-L256


# Vulnerability details




## Impact
If wxETH drips when nothing is staked, then the first staker can claim every drop.

## Proof of Concept
Suppose drip is enabled when `totalSupply() == 0`.
At least one block passes and the first staker stakes, just `1` xETH is enough. This mints her `1` wxETH. This also calls `_accrueDrip()` (by the `drip` modifier) which drips some amount of xETH. Note that `_accrueDrip()` is independent of `totalSupply()`, so it doesn't matter how little she stakes.
`cashMinusLocked` is now `1` plus the amount dripped.
Naturally, since she owns the entire supply she can immediately unstake the entire `cashMinusLocked`. Specifically, the `exchangeRate()` is now `(cashMinusLocked * BASE_UNIT) / totalSupply()` and she gets `(totalSupply() * exchangeRate()) / BASE_UNIT` = `cashMinusLocked`.

The issue is simply that drip is independent of staked amount, especially that it may drip even when nothing is staked, which enables the above attack.

## Recommended Mitigation Steps
Note what happens when `totalSupply() > 0`. Then drops will fall on existing wxETH, and any new staker will trigger a drip before having to pay the new exchange rate which includes the extra drops. So a new staker in this case cannot unstake more than they staked; all drops go to previous holders.
Therefore, do not drip when `totalSupply == 0`; in `_accrueDrip()`:
```diff
-if (!dripEnabled) return;
+if (!dripEnabled || totalSupply() == 0) return;
```






## Assessed type

MEV