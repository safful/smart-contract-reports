## Tags

- bug
- 3 (High Risk)
- primary issue
- satisfactory
- selected for report
- sponsor confirmed
- H-01

# [Re-balancing the vault allocation may always revert when distributing profits : resulting of a massive system DOS  ](https://github.com/code-423n4/2023-02-ethos-findings/issues/481) 

# Lines of code

https://github.com/code-423n4/2023-02-ethos/blob/73687f32b934c9d697b97745356cdf8a1f264955/Ethos-Core/contracts/StabilityPool.sol#L526-L538


# Vulnerability details

## Description

**[updateRewardSum](https://github.com/code-423n4/2023-02-ethos/blob/73687f32b934c9d697b97745356cdf8a1f264955/Ethos-Core/contracts/StabilityPool.sol#L486-L500)** function call **[_computeRewardsPerUnitStaked](https://github.com/code-423n4/2023-02-ethos/blob/73687f32b934c9d697b97745356cdf8a1f264955/Ethos-Core/contracts/StabilityPool.sol#L493)** with **_debtToOffset** set to 0. Meaning that the assignment [L531](https://github.com/code-423n4/2023-02-ethos/blob/73687f32b934c9d697b97745356cdf8a1f264955/Ethos-Core/contracts/StabilityPool.sol#L531) will revert if `lastLUSDLossError_Offset != 0` (which is likely the case) because we try to assign a negative value to an **uint**.

## Impact

**[_rebalance()](https://github.com/code-423n4/2023-02-ethos/blob/73687f32b934c9d697b97745356cdf8a1f264955/Ethos-Core/contracts/ActivePool.sol#L239)** will be definitely DOS if the profit is greater than the **[yieldClainThreshold](https://github.com/code-423n4/2023-02-ethos/blob/73687f32b934c9d697b97745356cdf8a1f264955/Ethos-Core/contracts/ActivePool.sol#L252)** ⇒ **[vars.profit != 0](https://github.com/code-423n4/2023-02-ethos/blob/73687f32b934c9d697b97745356cdf8a1f264955/Ethos-Core/contracts/ActivePool.sol#L286)**.

Because they call **_rebalance()** all these functions will be DOS :

In `BorrowerOperations` 100% DOS
    - openTrove
    - closeTrove
    - _adjustTrove
        - addColl, withdrawColl
        - withdrawLUSD, repayLUSD
In `TroveManager` 80% DOS
    - liquidateTroves
    - batchLiquidateTroves
    - redeemCloseTrove

## POC

Context : the vault has compound enough profit to withdraw. ([here](https://github.com/code-423n4/2023-02-ethos/blob/73687f32b934c9d697b97745356cdf8a1f264955/Ethos-Core/contracts/ActivePool.sol#L252))

Alice initiates a trove liquidation. **[offset()](https://github.com/code-423n4/2023-02-ethos/blob/73687f32b934c9d697b97745356cdf8a1f264955/Ethos-Core/contracts/StabilityPool.sol#L466)** in `StabilityPool` is called to cancels out the trove debt against the LUSD contained in the Stability Pool.

A floor division errors occur so now **[lastLUSDLossError_Offset](https://github.com/code-423n4/2023-02-ethos/blob/73687f32b934c9d697b97745356cdf8a1f264955/Ethos-Core/contracts/StabilityPool.sol#L537)** is not null. 

Now, every time **[_rebalance()](https://github.com/code-423n4/2023-02-ethos/blob/73687f32b934c9d697b97745356cdf8a1f264955/Ethos-Core/contracts/ActivePool.sol#L305)** is called the transaction will revert. 

## Mitigation

In [StabilityPool.sol#L504-L544](https://github.com/code-423n4/2023-02-ethos/blob/73687f32b934c9d697b97745356cdf8a1f264955/Ethos-Core/contracts/StabilityPool.sol#L504-L544), just skip the floor division errors calculation if `_debtToOffset == 0` 

```solidity
if(_debtToOffset != 0){
	[StabilityPool.sol#L526-L538](https://github.com/code-423n4/2023-02-ethos/blob/73687f32b934c9d697b97745356cdf8a1f264955/Ethos-Core/contracts/StabilityPool.sol#L526-L538)
}
```