## Tags

- bug
- 2 (Med Risk)
- primary issue
- selected for report
- sponsor acknowledged
- M-12

# [[NAZ-M1] Vault Fees Can Total To More Than `1e18`](https://github.com/code-423n4/2023-01-popcorn-findings/issues/540) 

# Lines of code

https://github.com/code-423n4/2023-01-popcorn//blob/main/src/vault/Vault.sol#L525


# Vulnerability details

## Impact
Currently in `Vault.sol` there are four different fee types:
1. Deposit
2. Withdrawal
3. Management
4. Performance

There is a proper check to make sure that individually none of them are `>= 1e18`. However, they can total to more than `1e18` and cause unsuspecting users to pay more than they may want to.

## Proof of Concept
Just taking the deposit and withdrawal fees into account say both of them are set to `5e17`, totaling to `1e18`. If a user were to than deposit and withdraw that would be 100%. Add in the other two fees and this situation gets even worse.

## Tools Used
Manual Review

## Recommended Mitigation Steps
Consider making sure that the total of all four fee types to be less than 1e18 instead of individually.