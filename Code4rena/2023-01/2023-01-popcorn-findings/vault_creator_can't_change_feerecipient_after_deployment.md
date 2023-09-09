## Tags

- bug
- 2 (Med Risk)
- primary issue
- selected for report
- sponsor confirmed
- M-31

# [Vault creator can't change feeRecipient after deployment](https://github.com/code-423n4/2023-01-popcorn-findings/issues/186) 

# Lines of code

https://github.com/code-423n4/2023-01-popcorn/blob/main/src/vault/Vault.sol#L553


# Vulnerability details

## Impact
The vault's `feeRecipient` can be updated through the `setFeeRecipient()` function. Only the owner of the contract (VaultController) can call it. But, the VaultController contract doesn't implement a function to call `setFeeRecipient()`. Thus, the function is actually not usable.

Since the vault creator won't be able to change the fee recipient they might potentially lose access to those funds.

## Proof of Concept
`setFeeRecipient` has the `onlyOwner` modifier which only allows the AdminProxy to access the function. The AdminProxy is called through the VaultController.
```sol
    function setFeeRecipient(address _feeRecipient) external onlyOwner {
        if (_feeRecipient == address(0)) revert InvalidFeeRecipient();

        emit FeeRecipientUpdated(feeRecipient, _feeRecipient);

        feeRecipient = _feeRecipient;
    }
```
The VaultController doesn't provide a function to execute `setFeeRecipient()`. Just search for `setFeeRecipient.selector` and you won't find anything.

## Tools Used
none

## Recommended Mitigation Steps
Add a function to interact with `setFeeRecipient()`.