## Tags

- bug
- 2 (Med Risk)
- primary issue
- satisfactory
- selected for report
- sponsor confirmed
- edited-by-warden
- M-23

# [The relation between the safe collateral ratio and the bad collateral ratio for the PeUSD vaults is not enforced correctly](https://github.com/code-423n4/2023-06-lybra-findings/issues/3) 

# Lines of code

https://github.com/code-423n4/2023-06-lybra/blob/26915a826c90eeb829863ec3851c3c785800594b/contracts/lybra/configuration/LybraConfigurator.sol#L127
https://github.com/code-423n4/2023-06-lybra/blob/26915a826c90eeb829863ec3851c3c785800594b/contracts/lybra/configuration/LybraConfigurator.sol#L202


# Vulnerability details

## Impact

The documentation states that: 

`The PeUSD vault requires a safe collateral rate at least 10% higher than the liquidation collateral rate, providing an additional buffer to protect against liquidation risks.`

Hence it is important to maintain the invariance between the relation of the safe collateral ratio (SCR) and the bad collateral ratio (BCR). Both functions `setSafeCollateralRatio` and `setBadCollateralRatio` at the `LybraConfigurator` contract make checks to ensure that the relation always holds.

The former is coded as:

```solidity
function setSafeCollateralRatio(address pool, uint256 newRatio) external checkRole(TIMELOCK) {
   if(IVault(pool).vaultType() == 0) {
      require(newRatio >= 160 * 1e18, "eUSD vault safe collateralRatio should more than 160%");
   } else {
      // @audit-ok SCR is always at least 10% greater than BCR.
      require(newRatio >= vaultBadCollateralRatio[pool] + 1e19, "PeUSD vault safe collateralRatio should more than bad collateralRatio");
     }

   vaultSafeCollateralRatio[pool] = newRatio;
   emit SafeCollateralRatioChanged(pool, newRatio);
}
```

The latter is coded as:

```solidity
function setBadCollateralRatio(address pool, uint256 newRatio) external onlyRole(DAO) {
  // @audit-issue BCR and SCR relationship is not enforced correctly.
  require(newRatio >= 130 * 1e18 && newRatio <= 150 * 1e18 && newRatio <= vaultSafeCollateralRatio[pool] + 1e19, "LNA");
  
  vaultBadCollateralRatio[pool] = newRatio;
  
  emit SafeCollateralRatioChanged(pool, newRatio);
}
```

We take only the logic clause related to the relationship between the BCR and SCR:

```solidity
require(newRatio <= vaultSafeCollateralRatio[pool] + 1e19);
```

We can see that the relationship is not coded correctly, we want the SCR always to be at least 10% higher than the BCR, so the correct check should be:

```solidity
require(newRatio <= vaultSafeCollateralRatio[pool] - 1e19);
```

## Proof of Concept

There is a path of actions that can lead to an SCR and a BCR that do not meet the requirement stated previously. For example:

- 1.- SCR is set to 150%
- 2.- BCR is also set to 150% (Incorrect requirement pass: 150% <= 150% + 10%)

## Tools Used

Manual Review

## Recommended Mitigation Steps

Change: 

```solidity
require(newRatio >= 130 * 1e18 && newRatio <= 150 * 1e18 && newRatio <= vaultSafeCollateralRatio[pool] + 1e19, "LNA");
```

to: 

```solidity
require(newRatio >= 130 * 1e18 && newRatio <= 150 * 1e18 && newRatio <= vaultSafeCollateralRatio[pool] - 1e19, "LNA");
```
















## Assessed type

Invalid Validation