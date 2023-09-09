## Tags

- bug
- 2 (Med Risk)
- selected for report
- sponsor confirmed
- M-30

# [Vault creator can't change quitPeriod](https://github.com/code-423n4/2023-01-popcorn-findings/issues/187) 

# Lines of code

https://github.com/code-423n4/2023-01-popcorn/blob/main/src/vault/Vault.sol#L629-L636


# Vulnerability details

## Impact
The vault's `quitPeriod` can be updated through the `setQuitPeriod()` function. Only the owner of the contract (AdminProxy through VaultController) can call it. But, the VaultController contract doesn't implement a function to call `setQuitPeriod()`. Thus, the function is actually not usable.

This limits the configuration of the vault. Every vault will have to use the standard 3-day `quitPeriod`. 

## Proof of Concept
`setQuitPeriod()` has the `onlyOwner` modifier which only allows the AdminProxy to access the function. The AdminProxy is called through the VaultController.
```sol
    function setQuitPeriod(uint256 _quitPeriod) external onlyOwner {
        if (_quitPeriod < 1 days || _quitPeriod > 7 days)
            revert InvalidQuitPeriod();

        quitPeriod = _quitPeriod;

        emit QuitPeriodSet(quitPeriod);
    }
```
The VaultController doesn't provide a function to execute `setQuitPeriod()`. Just search for `setQuidPeriod.selector` and you won't find anything.
## Tools Used
none

## Recommended Mitigation Steps
Add a function to interact with `setQuitPeriod()`.