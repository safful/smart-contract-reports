## Tags

- bug
- 2 (Med Risk)
- disagree with severity
- primary issue
- satisfactory
- selected for report
- sponsor disputed
- edited-by-warden
- M-15

# [Borrower and Supplier rewards  accrued could be lost when Admin replaces the reward distributor with a new reward distributor](https://github.com/code-423n4/2023-07-moonwell-findings/issues/58) 

# Lines of code

https://github.com/code-423n4/2023-07-moonwell/blob/fced18035107a345c31c9a9497d0da09105df4df/src/core/Comptroller.sol#L901-L909


# Vulnerability details

## Impact
Comptroller's admin user can replace the rewards distributor contract with a new MultiRewardDistributor 
using _setRewardDistributor() function.

At that time, if the supplier and borrowers had accured rewards that are not distributed. The _setRewardDistributor() function replaces the old reward distributor with the new one without disbursing the borrower and supplier rewards accured.

The new logic of computation and distribution could be different in the newer version and hence before replacement, the accured rewards should be distributed using the existing logic.

The new accruals could take effect with nil accruals providing a clean slate.

## Proof of Concept
The setRewardDistributor updates the mutiRewardDistributor contract with a new instance with out
transferring the accured rewards to borrowers and lenders.

```
    function _setRewardDistributor(MultiRewardDistributor newRewardDistributor) public {
        require(msg.sender == admin, "Unauthorized");
        MultiRewardDistributor oldRewardDistributor = rewardDistributor;
        rewardDistributor = newRewardDistributor;
        emit NewRewardDistributor(oldRewardDistributor, newRewardDistributor);
    }
```

The above set function should be added with a logic to look at all the markets and for each market,
the borrower and supplier rewards should be disbursed before the new instance is updated.
As long as there are accured rewards, the old instance should not be replaced.


## Tools Used
Manual review

## Recommended Mitigation Steps



Approach 1: Onchain approach
==========

step 1: Maintain a list of all accounts that participate in borrowing and supplying.
getAllParticipants() returns a list of participants.

Step 2: Add an internal function disburseAllRewards() which can be called by adminOnly.
This function looks for all partitipant accounts and for all markets, it claims rewards before updating the rewards distributor with a new instance.

```
 function disburseAllRewards() internal adminOnly {
        address[] memory holders = getAllParticipants();
        MToken[] memory mTokens = getAllMarkets();
        claimReward(holders, mTokens, true, true);
 }

 // @audit use the existing claimReward function to disburse the rewards to accounts that had accruals.

```

Approach 2: Offchain approach
==========
As the number of participants and markets grow, the onchain approach may result in DOS as the size of computations may not fit in a single block due to gas limit.

As an alternative, the claimReward() could be called from the outside for all the holders and markets
in smaller chunks.


Use getOutstandingRewardsForUser() function to check if there are any un-disbursed rewards in the MultiRewarddistributor contract. Only if there are no un-disbursed rewards, then only allow the admin to replace the reward distributor contract with a new instance.

Taking these steps will prevent the potential case where borrowers and suppliers could loose accured rewards.

Key drive away with this approach is, unless all the accured are distributed and they is nothing to distribute, dont update the multi reward distributor contract.

















## Assessed type

Token-Transfer