## Tags

- bug
- 2 (Med Risk)
- primary issue
- selected for report
- sponsor acknowledged
- M-01

# [Griefing attacks on `handleOps` and `multiSend` logic](https://github.com/code-423n4/2023-01-biconomy-findings/issues/499) 

# Lines of code

https://github.com/code-423n4/2023-01-biconomy/blob/5df2e8f8c0fd3393b9ecdad9ef356955f07fbbdd/scw-contracts/contracts/smart-contract-wallet/aa-4337/core/EntryPoint.sol#L68
https://github.com/code-423n4/2023-01-biconomy/blob/5df2e8f8c0fd3393b9ecdad9ef356955f07fbbdd/scw-contracts/contracts/smart-contract-wallet/libs/MultiSend.sol#L26


# Vulnerability details

## Description

The `handleOps` function executes an array of `UserOperation`. If at least one user operation fails the whole transaction will revert. That means the error on one user ops will fully reverts the other executed ops.

The `multiSend` function reverts if at least one of the transactions fails, so it is also vulnerable to such type of attacks.

## Attack scenario

Relayer offchain verify the batch of `UserOperation`s, convinced that they will receive fees, then send the `handleOps` transaction to the mempool. An attacker front-run the relayers transaction with another `handleOps` transaction that executes only one `UserOperation`, the last user operation from the relayers `handleOps` operations. An attacker will receive the funds for one `UserOperation`. Original relayers transaction will consume gas for the execution of all except one, user ops, but reverts at the end.

## Impact

Griefing attacks on the gas used for `handleOps` and `multiSend` function calls.

Please note, that while an attacker have no direct incentive to make such an attacks, they could short the token before the attack. 

## Recommended Mitigation Steps

Remove redundant `require`-like checks from internal functions called from the `handleOps` function and add the non-atomic execution logic to the `multiSend` function.