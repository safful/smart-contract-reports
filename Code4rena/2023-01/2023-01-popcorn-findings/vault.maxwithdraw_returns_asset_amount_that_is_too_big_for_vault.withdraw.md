## Tags

- bug
- 2 (Med Risk)
- selected for report
- sponsor acknowledged
- M-35

# [Vault.maxWithdraw returns asset amount that is too big for Vault.withdraw](https://github.com/code-423n4/2023-01-popcorn-findings/issues/67) 

# Lines of code

https://github.com/code-423n4/2023-01-popcorn/blob/main/src/vault/Vault.sol#L409-L411
https://github.com/code-423n4/2023-01-popcorn/blob/main/src/vault/Vault.sol#L211-L244
https://github.com/code-423n4/2023-01-popcorn/blob/main/src/vault/Vault.sol#L398-L416


# Vulnerability details

## Impact
ERC4626 standard requires `maxWithdraw` to return the maximum amount of underlying assets that can be withdrawn from the owner balance with a single withdraw call. `withdraw` in `Vault` implements a fee which is not calculated in `maxWithdraw` in `Vault`. Therefore, upstream contracts that call `maxWithdraw` and use the return value for withdraw will always revert.

 
 
## Proof of Concept

Upstream contract calls maxWithdraw to determine maximum amount of assets user can withdraw. `adapter` is the wrapper that allows interaction with the underlying ERC4626 protocol. 

```solidity
    function maxWithdraw(address caller) external view returns (uint256) {
        return adapter.maxWithdraw(caller);
    }
```

The upstream contract uses the return amount of assets as the input for `withdraw`. Notice that there is a withdrawal fee added by increasing the shares required. Since `maxWithdraw` would have returned the maximum assets that can be returned, increasing the shares required from withdrawal fee will mean that the shares required will exceed shares that user have. This will cause a revert during the transfer of shares from user to vault. 

```solidity
    function withdraw(
        uint256 assets,
        address receiver,
        address owner
    ) public nonReentrant syncFeeCheckpoint returns (uint256 shares) {
        if (receiver == address(0)) revert InvalidReceiver();


        shares = convertToShares(assets);


        uint256 withdrawalFee = uint256(fees.withdrawal);


        uint256 feeShares = shares.mulDiv(
            withdrawalFee,
            1e18 - withdrawalFee,
            Math.Rounding.Down
        );


        shares += feeShares;


        if (msg.sender != owner)
            _approve(owner, msg.sender, allowance(owner, msg.sender) - shares);


        _burn(owner, shares);


        if (feeShares > 0) _mint(feeRecipient, feeShares);


        adapter.withdraw(assets, receiver, address(this));


        emit Withdraw(msg.sender, receiver, owner, assets, shares);
    }


    function redeem(uint256 shares) external returns (uint256) {
        return redeem(shares, msg.sender, msg.sender);
    }
```

Note, this applies to all max* functions too in `Vault`. They all delegate to adapter which does not include withdrawal or deposit fee.

```solidity
    /// @return Maximum amount of underlying `asset` token that may be deposited for a given address. Delegates to adapter.
    function maxDeposit(address caller) public view returns (uint256) {
        return adapter.maxDeposit(caller);
    }

    /// @return Maximum amount of vault shares that may be minted to given address. Delegates to adapter.
    function maxMint(address caller) external view returns (uint256) {
        return adapter.maxMint(caller);
    }

    /// @return Maximum amount of underlying `asset` token that can be withdrawn by `caller` address. Delegates to adapter.
    function maxWithdraw(address caller) external view returns (uint256) {
        return adapter.maxWithdraw(caller);
    }

    /// @return Maximum amount of shares that may be redeemed by `caller` address. Delegates to adapter.
    function maxRedeem(address caller) external view returns (uint256) {
        return adapter.maxRedeem(caller);
    }

```


## Tools Used

Manual Review

## Recommended Mitigation Steps

Recommend calculating the withdrawal fee and deducting it in `maxWithdraw`. Same for the withdrawal and deposit fees for all the max* functions.


