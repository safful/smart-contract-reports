## Tags

- bug
- 2 (Med Risk)
- primary issue
- selected for report
- sponsor confirmed
- M-06

# [Flashloan fee is not distributed to the factory](https://github.com/code-423n4/2023-04-caviar-findings/issues/697) 

# Lines of code

https://github.com/code-423n4/2023-04-caviar/blob/main/src/PrivatePool.sol#L623-L654


# Vulnerability details

## Impact
Flashloan fee is not distributed to the factory

## Proof of Concept
When user takes a flashloan, then [he pays a fee](https://github.com/code-423n4/2023-04-caviar/blob/main/src/PrivatePool.sol#L651) to the PrivatePool.
The problem is that the whole fee amount is sent to PrivatePool and factory receives nothing.

However, all other function of contract send some part of fees to the factory.
For example, `change` function, which is similar to the `flashloan` as it doesn't change virtual nft and balance reserves. This function [calculates pool and protocol fees](https://github.com/code-423n4/2023-04-caviar/blob/main/src/PrivatePool.sol#L736-L737).

But in case of flashloan, only pool receives fees.
## Tools Used
VsCode
## Recommended Mitigation Steps
Send some part of flashloan fee to the factory.