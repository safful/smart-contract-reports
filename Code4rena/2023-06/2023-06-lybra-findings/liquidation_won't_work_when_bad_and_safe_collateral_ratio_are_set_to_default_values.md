## Tags

- bug
- 2 (Med Risk)
- downgraded by judge
- primary issue
- satisfactory
- selected for report
- sponsor confirmed
- M-21

# [Liquidation won't work when bad and safe collateral ratio are set to default values](https://github.com/code-423n4/2023-06-lybra-findings/issues/44) 

# Lines of code

https://github.com/code-423n4/2023-06-lybra/blob/5d70170f2c68dbd3f7b8c0c8fd6b0b2218784ea6/contracts/lybra/configuration/LybraConfigurator.sol#L339


# Vulnerability details

## Impact
`getBadCollateralRatio()` will revert because of underflow, if `vaultBadCollateralRatio[pool]` and `vaultSafeCollateralRatio[pool]` are set to 0 (i.e. using default ratios 150% and 130% accordingly).
It blocks liquidation flow.

## Proof of Concept
1e19 is decremented from value `vaultSafeCollateralRatio[pool]`
```solidity
    function getBadCollateralRatio(address pool) external view returns(uint256) {
        if(vaultBadCollateralRatio[pool] == 0) return vaultSafeCollateralRatio[pool] - 1e19;
        return vaultBadCollateralRatio[pool];
    }
```
However `vaultSafeCollateralRatio[pool]` can be set to 0, which should mean 160%:
```solidity
    function getSafeCollateralRatio(
        address pool
    ) external view returns (uint256) {
        if (vaultSafeCollateralRatio[pool] == 0) return 160 * 1e18;
        return vaultSafeCollateralRatio[pool];
    }
```
As the result incorrect accounting block liquidation when using default values.

Also I think this is similar issue, but different impact, therefore describe in this issue. BadCollateralRatio can't be set when SafeCollateralRatio is default, as newRatio must be less than 10%:
https://github.com/code-423n4/2023-06-lybra/blob/5d70170f2c68dbd3f7b8c0c8fd6b0b2218784ea6/contracts/lybra/configuration/LybraConfigurator.sol#L127
```solidity
    function setBadCollateralRatio(address pool, uint256 newRatio) external onlyRole(DAO) {
        require(newRatio >= 130 * 1e18 && newRatio <= 150 * 1e18 && newRatio <= vaultSafeCollateralRatio[pool] + 1e19, "LNA");
        ...
    }
```

## Tools Used
Manual Review

## Recommended Mitigation Steps
Instead of internal accessing variables, use functions `getSafeCollateralRatio()` and `getBadCollateralRatio()` in all the occurences, because variables can be zero.


## Assessed type

Invalid Validation