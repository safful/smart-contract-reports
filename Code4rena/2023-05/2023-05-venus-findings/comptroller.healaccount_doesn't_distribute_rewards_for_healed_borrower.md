## Tags

- bug
- 2 (Med Risk)
- satisfactory
- selected for report
- sponsor confirmed
- M-13

# [Comptroller.healAccount doesn't distribute rewards for healed borrower](https://github.com/code-423n4/2023-05-venus-findings/issues/116) 

# Lines of code

https://github.com/code-423n4/2023-05-venus/blob/main/contracts/Comptroller.sol#L578-L626


# Vulnerability details

## Impact
Comptroller.healAccount doesn't distribute rewards for healed borrower. As result healed account receives less rewards.

## Proof of Concept
`Comptroller.healAccount` can be called by anyone in order to fully close account. Healer should repay part of account's debt in order to receive all account's collateral. At the end account debt will be cleared.

This is the part when collateral is seized and debt is cleared.
https://github.com/code-423n4/2023-05-venus/blob/main/contracts/Comptroller.sol#L611-L625
```solidity
        for (uint256 i; i < userAssetsCount; ++i) {
            VToken market = userAssets[i];

            (uint256 tokens, uint256 borrowBalance, ) = _safeGetAccountSnapshot(market, user);
            uint256 repaymentAmount = mul_ScalarTruncate(percentage, borrowBalance);

            // Seize the entire collateral
            if (tokens != 0) {
                market.seize(liquidator, user, tokens);
            }
            // Repay a certain percentage of the borrow, forgive the rest
            if (borrowBalance != 0) {
                market.healBorrow(liquidator, user, repaymentAmount);
            }
        }
```

In order to seize collateral, `market.seize` is called, which will then [call `comptroller.preSeizeHook`](https://github.com/code-423n4/2023-05-venus/blob/main/contracts/VToken.sol#L1104). And this hook [will distribute supply rewards](https://github.com/code-423n4/2023-05-venus/blob/main/contracts/Comptroller.sol#L521-L525) to both accounts.

In order to clear healed account debt `market.healBorrow` is called. This function [will not call any comptroller function](https://github.com/code-423n4/2023-05-venus/blob/main/contracts/VToken.sol#L390-L429). As result, debt of healed account is set to 0, but rewards that were earned by account before healing were not distributed to him. So user lost rewards for that debt amount.

## Tools Used
VsCode
## Recommended Mitigation Steps
Comptroller should distribute rewards to this account that were earned before and only then set debt to 0.


## Assessed type

Other