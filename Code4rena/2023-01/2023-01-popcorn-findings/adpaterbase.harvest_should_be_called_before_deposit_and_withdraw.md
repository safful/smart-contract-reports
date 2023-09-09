## Tags

- bug
- 2 (Med Risk)
- primary issue
- selected for report
- sponsor acknowledged
- M-25

# [AdpaterBase.harvest should be called before deposit and withdraw](https://github.com/code-423n4/2023-01-popcorn-findings/issues/302) 

# Lines of code

https://github.com/code-423n4/2023-01-popcorn/blob/main/src/vault/adapter/abstracts/AdapterBase.sol#L438-L450
https://github.com/code-423n4/2023-01-popcorn/blob/main/src/vault/adapter/abstracts/AdapterBase.sol#L162
https://github.com/code-423n4/2023-01-popcorn/blob/main/src/vault/adapter/abstracts/AdapterBase.sol#L232


# Vulnerability details

## Impact
AdpaterBase.harvest should be called before deposit and withdraw.
## Proof of Concept
Function `harvest` is called in order to receive yields for the adapter. It calls strategy, which handles that process. Depending on strategy it can call `strategyDeposit` function in order to [deposit earned amount](https://github.com/code-423n4/2023-01-popcorn/blob/main/src/vault/adapter/abstracts/AdapterBase.sol#L456-L461) through the adaptor.
That actually means that in case if totalAssets was X before `harvest` call, then after it becomes X+Y, in case if Y yields were earned by adapter and strategy deposited it. So for the same amount of shares, user can receive bigger amount of assets.
 
When user deposits or withdraw then `harvest` function [is called](https://github.com/code-423n4/2023-01-popcorn/blob/main/src/vault/adapter/abstracts/AdapterBase.sol#L232), but it's called after shares amount calculation.
Because of that in case of deposit, all previous depositors lose some part of yields as they share it with new depositor.
And in case of withdraw, user loses his yields.
## Tools Used
VsCode
## Recommended Mitigation Steps
Call `harvest` before shares amount calculation.