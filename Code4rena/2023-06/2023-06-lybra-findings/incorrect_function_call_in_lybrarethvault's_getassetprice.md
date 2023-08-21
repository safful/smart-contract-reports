## Tags

- bug
- 2 (Med Risk)
- primary issue
- satisfactory
- selected for report
- sponsor confirmed
- edited-by-warden
- M-22

# [Incorrect function call in LybraRETHVault's getAssetPrice](https://github.com/code-423n4/2023-06-lybra-findings/issues/27) 

# Lines of code

https://github.com/code-423n4/2023-06-lybra/blob/5d70170f2c68dbd3f7b8c0c8fd6b0b2218784ea6/contracts/lybra/pools/LybraRETHVault.sol#L46-L48


# Vulnerability details

## Impact

The incorrect function call in the code results in the inability to calculate the asset price properly. This will halt all operations associated with the asset pricing, disrupting the functioning of the entire system.


## Proof of Concept

LybraRETHVault's `getAssetPrice` method currently makes a call to a non-existent function in the rETH contract, `getExchangeRatio()`. The issue appears to be a misunderstanding or miscommunication, as the rETH contract does not provide a `getExchangeRatio()` function. This leads to a failure in the asset price calculation.

```solidity
    function getAssetPrice() public override returns (uint256) {
        return (_etherPrice() * IRETH(address(collateralAsset)).getExchangeRatio()) / 1e18;
    }
```

The correct function to call is `getExchangeRate()`, which exists in the rETH contract and provides the exchange rate necessary to determine the asset price.


https://etherscan.deth.net/address/0xae78736Cd615f374D3085123A210448E74Fc6393
![Uploading file...ud2r6]()

![](https://i.imgur.com/emneslX.png)

https://etherscan.deth.net/address/0xae78736Cd615f374D3085123A210448E74Fc6393
![](https://i.imgur.com/qk9Ae0y.png)


## Tools Used

Manual review


## Recommended Mitigation Steps

To resolve this issue, it is recommended to replace the non-existent function call `getExchangeRatio()` with the correct function `getExchangeRate()`. This correction will ensure that the `getAssetPrice()` method retrieves the correct exchange rate from the rETH contract, allowing the system to calculate the asset price accurately.

```solidity
    function getAssetPrice() public override returns (uint256) {
        return (_etherPrice() * IRETH(address(collateralAsset)).getExchangeRate()) / 1e18;
    }
```








## Assessed type

Other