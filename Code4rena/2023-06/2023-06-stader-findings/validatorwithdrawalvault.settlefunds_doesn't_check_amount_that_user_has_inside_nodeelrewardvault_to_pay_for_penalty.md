## Tags

- bug
- 2 (Med Risk)
- satisfactory
- selected for report
- sponsor acknowledged
- M-12

# [ValidatorWithdrawalVault.settleFunds doesn't check amount that user has inside NodeELRewardVault to pay for penalty](https://github.com/code-423n4/2023-06-stader-findings/issues/84) 

# Lines of code

https://github.com/code-423n4/2023-06-stader/blob/main/contracts/ValidatorWithdrawalVault.sol#L53-L83


# Vulnerability details

## Impact
ValidatorWithdrawalVault.settleFunds doesn't check amount that user has inside NodeELRewardVault to pay for penalty. That value can increase operator's earned amount which can avoid slashing for him.

## Proof of Concept
When validator withdraws from beacon chain then `ValidatorWithdrawalVault.settleFunds` function is called. This function calculates amount that validator [has earned](https://github.com/code-423n4/2023-06-stader/blob/main/contracts/ValidatorWithdrawalVault.sol#L62) for attestations as validator. So only balance of this contract [is considered](https://github.com/code-423n4/2023-06-stader/blob/main/contracts/ValidatorWithdrawalVault.sol#L99).

Then function [fetches penalty amount](https://github.com/code-423n4/2023-06-stader/blob/main/contracts/ValidatorWithdrawalVault.sol#L64). This penalty amount contains [of 3 points](https://github.com/code-423n4/2023-06-stader/blob/main/contracts/Penalty.sol#L111): _mevTheftPenalty, _missedAttestationPenalty and _missedAttestationPenalty.

In case if penalty amount is bigger than validator's earning on `ValidatorWithdrawalVault`, then [SD collateral is slashed](https://github.com/code-423n4/2023-06-stader/blob/main/contracts/ValidatorWithdrawalVault.sol#L66-L69).

Now we need to understand how validator receives funds in this system. All attestation payments comes to `ValidatorWithdrawalVault`, while mev/block proposal funds are coming to `SocializingPool` or `NodeELRewardVault`(depends on user's choice). So actually `_missedAttestationPenalty` is responding to `ValidatorWithdrawalVault` earning, while `_mevTheftPenalty` is responding to `NodeELRewardVault` earnings.

That means that `NodeELRewardVault` balance should also be checked in order to find out how mane earning validator has there and they should be also counted when applying penalty.

Simple example.
1.Validator wants to exit.
2.Operator earning is 0.1 eth inside `ValidatorWithdrawalVault`.
3.Accrued penalty is 0.11, which means that user will be slashed.
4.Operator also has `NodeELRewardVault` where his operator's reward is 0.05 eth
5.As result user had enough balance to cover penalty, but he still was penalizied.
## Tools Used
VsCode
## Recommended Mitigation Steps
As you accrued `_mevTheftPenalty` inside `ValidatorWithdrawalVault`, then you also should calculate operator's rewards inside `NodeELRewardVault`.


## Assessed type

Error