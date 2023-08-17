## Tags

- bug
- 2 (Med Risk)
- primary issue
- selected for report
- M-01

# [A staker with verified over-commitment can potentially bypass slashing completely](https://github.com/code-423n4/2023-04-eigenlayer-findings/issues/408) 

# Lines of code

https://github.com/code-423n4/2023-04-eigenlayer/blob/5e4872358cd2bda1936c29f460ece2308af4def6/src/contracts/core/StrategyManager.sol#L197
https://github.com/code-423n4/2023-04-eigenlayer/blob/5e4872358cd2bda1936c29f460ece2308af4def6/src/contracts/core/StrategyManager.sol#L513
https://github.com/code-423n4/2023-04-eigenlayer/blob/5e4872358cd2bda1936c29f460ece2308af4def6/src/contracts/pods/EigenPod.sol#L293
https://github.com/code-423n4/2023-04-eigenlayer/blob/5e4872358cd2bda1936c29f460ece2308af4def6/src/contracts/pods/EigenPod.sol#L396


# Vulnerability details

## Description
In EigenLayer, watchers submit over-commitment proof in the event a staker's balance on the Beacon chain falls below the minimum restaked amount per validator. In such a scenario, stakers’ shares are [decreased by the restaked amount](https://github.com/code-423n4/2023-04-eigenlayer/blob/5e4872358cd2bda1936c29f460ece2308af4def6/src/contracts/pods/EigenPod.sol#L293). Note that when a full withdrawal is processed, stakers’ deducted shares are [credited back to allow for a planned withdrawal on EigenLayer](https://github.com/code-423n4/2023-04-eigenlayer/blob/5e4872358cd2bda1936c29f460ece2308af4def6/src/contracts/pods/EigenPod.sol#L396)

If such a staker has delegated to an operator who gets slashed on EigenLayer, there is a possibility that this staker completely bypasses any slashing penalties. If overcommitment reduced the shares in the stakers account to 0, there is nothing available for governance to slash.

It is reasonable to assume that governance calls `StrategyManager::slashShares` to reduce a percentage of shares (penalty) from all stakers who delegated to a slashed operator and then resets the frozen status of operator by calling `ISlasher::resetFrozenStatus`.

By simply unstaking on the beacon chain AFTER slashing is complete (and operator is unfrozen), an over-committed staker can simply unstake on beacon chain and get back their shares that were deducted when over-commitment was recorded (refer to PoC). Note that these shares have not faced any slashing penalties.

## Impact
An over-committed staker can avoid being slashed in a scenario in which their stake should be subject to slashing and so we evaluate the severity to **MEDIUM**.

## Proof of Concept
Consider the following scenario with a chain of events in this order:

1. Alice stakes 32 ETH and gets corresponding shares in `BeaconEthStrategy`.
2. After some time, Alice gets slashed on the BeaconChain and her current balance on the Beacon Chain is now less than what she restaked on EigenLayer.
3. An observer will submit a proof via `EigenPod::verifyOverCommittedStake` and Alice's shares are now decreased to 0 (Note: this will be credited back to Alice when she withdraws from the Beacon Chain using `EigenPod::verifyAndProcessWithdrawal`)
4. Next, Alice's operator gets slashed and her account gets frozen.
5. Governance slashes Alice along with all stakers who delegated to slashed operator (there is nothing to slash since Alice's shares are currently 0).
6. After slashing everyone, governance resets frozen status of operator by calling `ISlasher::resetFrozenStatus`.
7. Alice now unstakes on the Beacon Chain and gets a credit of shares that were earlier deducted while recording over-commitment.
8. Alice queues a withdrawal request and completes withdrawal without facing any slashing penalty.

## Tools Used
Manual review

## Recommended Mitigation Steps
Over-commitment needs to be accounted for when slashing such that a staker is being slashed not just shares in their account, but also the amount that is temporarily debited while recording over-commitment.

Consider adding a mapping to `StrategyManager` that keeps track of over-commitment for each staker and take that into consideration while slashing in `StrategyManager::slashShares`.


## Assessed type

Other