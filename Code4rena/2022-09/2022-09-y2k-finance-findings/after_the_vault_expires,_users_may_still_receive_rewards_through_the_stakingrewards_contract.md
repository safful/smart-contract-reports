## Tags

- bug
- 2 (Med Risk)
- sponsor disputed
- selected for report

# [After the vault expires, users may still receive rewards through the StakingRewards contract](https://github.com/code-423n4/2022-09-y2k-finance-findings/issues/57) 

# Lines of code

https://github.com/code-423n4/2022-09-y2k-finance/blob/ac3e86f07bc2f1f51148d2265cc897e8b494adf7/src/Controller.sol#L223


# Vulnerability details

## Impact
When the triggerEndEpoch function of the Controller contract is called, the assets in the insrVault will be sent to the riskVault, which also means that the tokens in the insrVault will be worthless. 
```
insrVault.setClaimTVL(epochEnd, 0);
...
        else {
            entitledAmount = amount.divWadDown(idFinalTVL[id]).mulDivDown(
                idClaimTVL[id],
                1 ether
            );
        }
```
However, if the periodFinish > _epochEnd in the StakingRewards contract corresponding to the insrVault, the user can continue to stake his insrToken and receive rewards.
## Proof of Concept
https://github.com/code-423n4/2022-09-y2k-finance/blob/ac3e86f07bc2f1f51148d2265cc897e8b494adf7/src/Controller.sol#L223
https://github.com/code-423n4/2022-09-y2k-finance/blob/ac3e86f07bc2f1f51148d2265cc897e8b494adf7/src/Vault.sol#L418-L423
https://github.com/code-423n4/2022-09-y2k-finance/blob/ac3e86f07bc2f1f51148d2265cc897e8b494adf7/src/rewards/StakingRewards.sol#L183-L210
## Tools Used
None
## Recommended Mitigation Steps
Add the following code at the end of the notifyRewardAmount function of the StakingRewards contract to limit the periodFinish
```
         lastUpdateTime = block.timestamp;
         periodFinish = block.timestamp.add(rewardsDuration);
+       require(periodFinish <= id);
```