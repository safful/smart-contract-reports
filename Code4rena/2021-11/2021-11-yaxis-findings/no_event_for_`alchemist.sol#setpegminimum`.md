## Tags

- bug
- 0 (Non-critical)
- disagree with severity
- sponsor confirmed
- wont fix

# [No event for `Alchemist.sol#setPegMinimum`](https://github.com/code-423n4/2021-11-yaxis-findings/issues/24) 

# Handle

0x0x0x


# Vulnerability details

## Impact
The change of `pegMinimum` is crucial for the funcionality of the contract. Users should be informed about the changes. Furthermore, when `pegMinimum` is set to be maximum of `uin256`, functions such as `mint`, `liquidate` and `repay` cannot be used. Therefore, the change of `pegMinimum` should be emitted to create a safe environment for users. 

## Tools Used
Manual analysis
## Recommended Mitigation Steps
Emit the changes. Furthermore, it would be better if for such a change users get notified beforehand with a mechanism such as Timelock.

