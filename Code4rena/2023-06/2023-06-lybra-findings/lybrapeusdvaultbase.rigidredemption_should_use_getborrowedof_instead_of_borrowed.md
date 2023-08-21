## Tags

- bug
- 2 (Med Risk)
- primary issue
- satisfactory
- selected for report
- sponsor confirmed
- M-08

# [LybraPeUSDVaultBase.rigidRedemption should use getBorrowedOf instead of borrowed](https://github.com/code-423n4/2023-06-lybra-findings/issues/679) 

# Lines of code

https://github.com/code-423n4/2023-06-lybra/blob/5d70170f2c68dbd3f7b8c0c8fd6b0b2218784ea6/contracts/lybra/pools/base/LybraPeUSDVaultBase.sol#L157-L168


# Vulnerability details

## Impact
In LybraPeUSDVaultBase, the return value of getBorrowedOf represents the user's debt, while borrowed only represents the user's borrowed funds and does not include fees.
Using borrowed instead of getBorrowedOf in rigidRedemption results in
1. the requirement for the peusdAmount parameter is smaller than it actually is
2. the calculated providerCollateralRatio is larger, so that rigidRedemption can be performed even if the actual providerCollateralRatio is less than 100e18.
```solidity
    function rigidRedemption(address provider, uint256 peusdAmount) external virtual {
        require(configurator.isRedemptionProvider(provider), "provider is not a RedemptionProvider");
        require(borrowed[provider] >= peusdAmount, "peusdAmount cannot surpass providers debt");
        uint256 assetPrice = getAssetPrice();
        uint256 providerCollateralRatio = (depositedAsset[provider] * assetPrice * 100) / borrowed[provider];
        require(providerCollateralRatio >= 100 * 1e18, "provider's collateral ratio should more than 100%");
        _repay(msg.sender, provider, peusdAmount);
        uint256 collateralAmount = (((peusdAmount * 1e18) / assetPrice) * (10000 - configurator.redemptionFee())) / 10000;
        depositedAsset[provider] -= collateralAmount;
        collateralAsset.transfer(msg.sender, collateralAmount);
        emit RigidRedemption(msg.sender, provider, peusdAmount, collateralAmount, block.timestamp);
    }
```
## Proof of Concept
https://github.com/code-423n4/2023-06-lybra/blob/5d70170f2c68dbd3f7b8c0c8fd6b0b2218784ea6/contracts/lybra/pools/base/LybraPeUSDVaultBase.sol#L157-L168
## Tools Used
None
## Recommended Mitigation Steps
Change to
```diff
    function rigidRedemption(address provider, uint256 peusdAmount) external virtual {
        require(configurator.isRedemptionProvider(provider), "provider is not a RedemptionProvider");
-       require(borrowed[provider] >= peusdAmount, "peusdAmount cannot surpass providers debt");
+       require(getBorrowedOf(provider) >= peusdAmount, "peusdAmount cannot surpass providers debt");
        uint256 assetPrice = getAssetPrice();
-       uint256 providerCollateralRatio = (depositedAsset[provider] * assetPrice * 100) / borrowed[provider];
+       uint256 providerCollateralRatio = (depositedAsset[provider] * assetPrice * 100) / getBorrowedOf(provider);
        require(providerCollateralRatio >= 100 * 1e18, "provider's collateral ratio should more than 100%");
        _repay(msg.sender, provider, peusdAmount);
        uint256 collateralAmount = (((peusdAmount * 1e18) / assetPrice) * (10000 - configurator.redemptionFee())) / 10000;
        depositedAsset[provider] -= collateralAmount;
        collateralAsset.transfer(msg.sender, collateralAmount);
        emit RigidRedemption(msg.sender, provider, peusdAmount, collateralAmount, block.timestamp);
    }
```


## Assessed type

Error