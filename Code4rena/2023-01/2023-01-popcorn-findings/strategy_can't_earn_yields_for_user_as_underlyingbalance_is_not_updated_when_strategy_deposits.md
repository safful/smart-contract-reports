## Tags

- bug
- 2 (Med Risk)
- downgraded by judge
- satisfactory
- selected for report
- sponsor acknowledged
- M-18

# [Strategy can't earn yields for user as underlyingBalance is not updated when strategy deposits](https://github.com/code-423n4/2023-01-popcorn-findings/issues/467) 

# Lines of code

https://github.com/code-423n4/2023-01-popcorn/blob/main/src/vault/adapter/abstracts/AdapterBase.sol#L158
https://github.com/code-423n4/2023-01-popcorn/blob/main/src/vault/adapter/abstracts/AdapterBase.sol#L456-L472


# Vulnerability details

## Impact
Strategy can't earn yields for user as underlyingBalance is not updated when strategy deposits.

## Proof of Concept
When someone deposits/withdraws from adapter, then `underlyingBalance` variable [is updated](https://github.com/code-423n4/2023-01-popcorn/blob/main/src/vault/adapter/abstracts/AdapterBase.sol#L158) with deposited/withdrawn to the vault shares amount.
Only when user deposits or withdraws, then `AdapterBase` [changes totalSupply](https://github.com/code-423n4/2023-01-popcorn/blob/main/src/vault/adapter/abstracts/AdapterBase.sol#L160)(it mints or burns shares).

This is how shares of users are calculated inside BeffyAdapter
https://github.com/code-423n4/2023-01-popcorn/blob/main/src/vault/adapter/beefy/BeefyAdapter.sol#L122-L133
```solidity
    function convertToUnderlyingShares(uint256, uint256 shares)
        public
        view
        override
        returns (uint256)
    {
        uint256 supply = totalSupply();
        return
            supply == 0
                ? shares
                : shares.mulDiv(underlyingBalance, supply, Math.Rounding.Up);
    }
```
As you can see, when user provides amount of shares that he wants to withdraw, then these shares are recalculated in order to receive shares amount inside vault. This depends on underlyingBalance and totalSupply.

Each adapter can have a strategy that can withdraw `harvest` and then redeposit it inside the vault. In this case users should earn new shares.
When adapter wants to deposit to vault it should call `strategyDeposit` function.
https://github.com/code-423n4/2023-01-popcorn/blob/main/src/vault/adapter/abstracts/AdapterBase.sol#L456-L461
```solidity
    function strategyDeposit(uint256 amount, uint256 shares)
        public
        onlyStrategy
    {
        _protocolDeposit(amount, shares);
    }
```
This function is just sending all amount to the vault.
But actually it should also increase `underlyingBalance` with shares amount that it will receive by depositing.
Because this is not happening, `underlyingBalance` always equal to totalSupply and users do not earn any yields using strategy as `convertToUnderlyingShares` function will just return same value as provided shares.
So currently users can't earn any yields using strategy.

Note: this was discussed with protocol developer and he explained me how it should work.
## Tools Used
VsCode
## Recommended Mitigation Steps
Increase `underlyingBalance` with shares amount that it will receive by depositing. But do not mint shares)