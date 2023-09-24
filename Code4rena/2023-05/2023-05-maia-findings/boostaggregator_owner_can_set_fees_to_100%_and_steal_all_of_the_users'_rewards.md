## Tags

- bug
- 2 (Med Risk)
- downgraded by judge
- primary issue
- satisfactory
- selected for report
- sponsor confirmed
- M-14

# [BoostAggregator owner can set fees to 100% and steal all of the users' rewards](https://github.com/code-423n4/2023-05-maia-findings/issues/634) 

# Lines of code

https://github.com/code-423n4/2023-05-maia/blob/main/src/talos/boost-aggregator/BoostAggregator.sol#L119
https://github.com/code-423n4/2023-05-maia/blob/main/src/talos/boost-aggregator/BoostAggregator.sol#L153


# Vulnerability details

## Impact
Users who use `BoostAggregator` will suffer 100% loss of their rewards.

## Proof of Concept
After users have staked their tokens, the owner of the `BoostAggregator` can set `protocolFee` to 10 000 (100%) and steal of the users' rewards. Anyone can create their own `BoostAggregator` and it is supposed to be publicly used, therefore the owner of it cannot be considered trusted. Allowing the owner to steal users' rewards is an unnecessary vulnerability.

```solidity
    function setProtocolFee(uint256 _protocolFee) external onlyOwner { 
        if (_protocolFee > DIVISIONER) revert FeeTooHigh();
        protocolFee = _protocolFee; // @audit - owner can set it to 100% and steal all rewards
    }
```

## Tools Used
Manual review

## Recommended Mitigation Steps
Create a mapping which tracks the `protocolFee` at which the user has deposited their NFT, upon withdrawing get the protocolFee from the said mapping.


## Assessed type

Other