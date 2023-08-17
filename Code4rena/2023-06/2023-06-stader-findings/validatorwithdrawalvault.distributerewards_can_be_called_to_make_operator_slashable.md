## Tags

- bug
- 2 (Med Risk)
- satisfactory
- selected for report
- sponsor disputed
- M-11

# [ValidatorWithdrawalVault.distributeRewards can be called to make operator slashable](https://github.com/code-423n4/2023-06-stader-findings/issues/86) 

# Lines of code

https://github.com/code-423n4/2023-06-stader/blob/main/contracts/ValidatorWithdrawalVault.sol#L29-L83


# Vulnerability details

## Impact
ValidatorWithdrawalVault.distributeRewards can be called to make operator slashable. Attacker can call `distributeRewards` right before `settleFunds` to make `operatorShare < penaltyAmount`. As result validator will face loses.

## Proof of Concept
`ValidatorWithdrawalVault.distributeRewards` can be called by anyone. It's purpose is to distribute validators rewards among stakers protocol and operator. After the call, balance of `ValidatorWithdrawalVault` becomes 0.

`ValidatorWithdrawalVault.settle` is called when validator is withdrawn from beacon chain. In this case balance of contract is used to [find operatorShare](https://github.com/code-423n4/2023-06-stader/blob/main/contracts/ValidatorWithdrawalVault.sol#L62) and in case if it's less than accrued penalty by validator, then operator is slashed.

Because  `distributeRewards` is permissionless, then next situation is possible.
1.Operator decided to withdraw validator. At the moment of that call, balance of `ValidatorWithdrawalVault` is not 0 and operatorShare is 1 eth. Also validator accrued 4.5 eth of penalty.
2.Malicious user sees when 32 eth of validator's deposit is sent to the `ValidatorWithdrawalVault` and frontruns it with `distributeRewards` call. This makes balance to be 32 eth.
3.operatorShare will be 4 eth in this time(permisssionless) and penalty is 4.5, so user is slashed.
4.In case if malicious user didn't call `distributeRewards`, then slash would not occur.

Also in same way, permissioned operator can call `distributeRewards` to get his rewards, when he is going to be slashed. As permissioned validators are not forced to have collateral to be slashed. By this move he rescued his eraning as otherwise they would be sent to pool.
## Tools Used
VsCode
## Recommended Mitigation Steps
Maybe think about restrict access to `distributeRewards` function.


## Assessed type

Access Control