## Tags

- bug
- 3 (High Risk)
- primary issue
- satisfactory
- selected for report
- sponsor confirmed
- H-02

# [doesn't calculate the current borrowing amount for the provider, including the provider's borrowed shares and accumulated fees due to Inconsistency in collateralRatio calculation](https://github.com/code-423n4/2023-06-lybra-findings/issues/723) 

# Lines of code

https://github.com/code-423n4/2023-06-lybra/blob/main/contracts/lybra/pools/base/LybraPeUSDVaultBase.sol#L127
https://github.com/code-423n4/2023-06-lybra/blob/main/contracts/lybra/pools/base/LybraPeUSDVaultBase.sol#L161


# Vulnerability details

## Impact
doesn't calculate the current borrowing amount for the provider, including the provider's borrowed shares and accumulated fees.

## Proof of Concept
Borrowers collateralRatio in the liquidation() function is calculated by:
```solidity
uint256 onBehalfOfCollateralRatio = (depositedAsset[onBehalfOf] * assetPrice * 100) / getBorrowedOf(onBehalfOf);
```
notice it calls the *getBorrowedOf()* function, which
calculate the current borrowing amount for the borrower, including the borrowed shares and accumulated fees and not just the borrowed amount
https://github.com/code-423n4/2023-06-lybra/blob/main/contracts/lybra/pools/base/LybraPeUSDVaultBase.sol#L253
```solidity
function getBorrowedOf(address user) public view returns (uint256) {
        return borrowed[user] + feeStored[user] + _newFee(user);
    }
```
However, the providers collateralRatio in the rigidRedemption() function is calculated by:
https://github.com/code-423n4/2023-06-lybra/blob/main/contracts/lybra/pools/base/LybraPeUSDVaultBase.sol#L161
```solidity
uint256 providerCollateralRatio = (depositedAsset[provider] * assetPrice * 100) / borrowed[provider];
```
here the deposit asset is divided by just the borrowed amount, missing out the borrowed shares and accumulated fees
## Tools Used
Visual Studio Code

## Recommended Mitigation Steps
Be consistent with collateralRatio calculation


## Assessed type

Other