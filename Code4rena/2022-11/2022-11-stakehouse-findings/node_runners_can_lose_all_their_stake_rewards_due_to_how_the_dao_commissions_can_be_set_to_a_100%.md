## Tags

- bug
- 2 (Med Risk)
- primary issue
- satisfactory
- selected for report
- sponsor disputed
- M-18

# [Node runners can lose all their stake rewards due to how the DAO commissions can be set to a 100%](https://github.com/code-423n4/2022-11-stakehouse-findings/issues/190) 

# Lines of code

https://github.com/code-423n4/2022-11-stakehouse/blob/4b6828e9c807f2f7c569e6d721ca1289f7cf7112/contracts/liquid-staking/LiquidStakingManager.sol#L948-L955


# Vulnerability details

## Impact
Node runners can have all their stake rewards taken by the DAO as commissions can be set to a 100%.

## Proof of Concept
There is no limits on `_updateDAORevenueCommission()` except not exceeding `MODULO`, which means it can be set to a 100%.

[LiquidStakingManager.sol#L948-L955](https://github.com/code-423n4/2022-11-stakehouse/blob/4b6828e9c807f2f7c569e6d721ca1289f7cf7112/contracts/liquid-staking/LiquidStakingManager.sol#L948-L955)
```solidity
    function _updateDAORevenueCommission(uint256 _commissionPercentage) internal {
        require(_commissionPercentage <= MODULO, "Invalid commission");

        emit DAOCommissionUpdated(daoCommissionPercentage, _commissionPercentage);

        daoCommissionPercentage = _commissionPercentage;
    }
```
This percentage is used to calculate `uint256 daoAmount = (_received * daoCommissionPercentage) / MODULO` in `_calculateCommission()`.
Remaining is then caculated with `uint256 rest = _received - daoAmount`, and in this case `rest = 0`.
When node runner calls `claimRewardsAsNodeRunner()`, the node runner will receive 0 rewards.

## Tools Used
Manual Review

## Recommended Mitigation Steps
There should be maximum cap on how much commission DAO can take from node runners.