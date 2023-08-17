## Tags

- bug
- 2 (Med Risk)
- sponsor disputed
- selected for report

# [It's possible to change for Vault and lost control on it](https://github.com/code-423n4/2022-09-y2k-finance-findings/issues/66) 

# Lines of code

https://github.com/code-423n4/2022-09-y2k-finance/blob/main/src/VaultFactory.sol#L345-L359
https://github.com/code-423n4/2022-09-y2k-finance/blob/main/src/Controller.sol#L136
https://github.com/code-423n4/2022-09-y2k-finance/blob/main/src/Controller.sol#L152
https://github.com/code-423n4/2022-09-y2k-finance/blob/main/src/VaultFactory.sol#L187-L190


# Vulnerability details

## Impact
`VaultFactory` allows admin to change `controller` for marketId(hedge and risk vaults) using `VaultFactory.changeController`. This method then set controller to both vaults. This address is important for `Vault` contract as it allows to call different functions.

`VaultFactory` take care about different pair vaults through `indexVaults` mapping. `Controller` can get info about pairs vaults only through the correct `VaultFactory` that is provided to `Controller` in constructor.

It's possible that `VaultFactory.changeController` will set controller whose `vaultFactory` field is not equal to current `VaultFactory`. That means that when `Controller.triggerDepeg` or `Controller.triggerEndEpoch` will be called they will not be able to find the market.
So current controller will not be able to call hedge and risk vaults.

## Proof of Concept
This is how the `controller` is set to vaults.
https://github.com/code-423n4/2022-09-y2k-finance/blob/main/src/VaultFactory.sol#L345-L359

Controller depends on `VaultFactory` to find vault for market.
https://github.com/code-423n4/2022-09-y2k-finance/blob/main/src/Controller.sol#L136
https://github.com/code-423n4/2022-09-y2k-finance/blob/main/src/Controller.sol#L152


## Tools Used

## Recommended Mitigation Steps
Use same check as you used in `VaultFactory.createNewMarket`
https://github.com/code-423n4/2022-09-y2k-finance/blob/main/src/VaultFactory.sol#L187-L190