## Tags

- bug
- 2 (Med Risk)
- satisfactory
- sponsor acknowledged
- edited-by-warden
- selected for report
- M-01

# [Due to loss of precision, targetVotes may not reach](https://github.com/code-423n4/2022-10-paladin-findings/issues/59) 

# Lines of code

https://github.com/code-423n4/2022-10-paladin/blob/d6d0c0e57ad80f15e9691086c9c7270d4ccfe0e6/contracts/WardenPledge.sol#L245-L246


# Vulnerability details

## Impact
In the _pledge function, require delegationBoost.adjusted_balance_of(pledgeParams.receiver) + amount <= pledgeParams.targetVotes.
In reality, when the user pledges amount votes, the actual votes received by the receiver are the bias in the following calculation. And the bias will be less than amount due to the loss of precision.
```
        uint256 slope = amount / boostDuration;
        uint256 bias = slope * boostDuration;
```
This means that the balance of receiver may not reach targetVotes
```
    point = self._checkpoint_read(_user, False)
    amount += (point.bias - point.slope * (block.timestamp - point.ts))
    return amount
```
## Proof of Concept
https://github.com/code-423n4/2022-10-paladin/blob/d6d0c0e57ad80f15e9691086c9c7270d4ccfe0e6/contracts/WardenPledge.sol#L245-L246
https://github.com/curvefi/curve-veBoost/blob/master/contracts/BoostV2.vy#L192-L209
https://github.com/curvefi/curve-veBoost/blob/master/contracts/BoostV2.vy#L175
## Tools Used
None
## Recommended Mitigation Steps
Use bias instead of amount in the check below
```
        uint256 slope = amount / boostDuration;
        uint256 bias = slope * boostDuration;
        if(delegationBoost.adjusted_balance_of(pledgeParams.receiver) + bias > pledgeParams.targetVotes) revert Errors.TargetVotesOverflow();
        delegationBoost.boost(
            pledgeParams.receiver,
            amount,
            endTimestamp,
            user
        );
```