## Tags

- bug
- 1 (Low Risk)
- sponsor confirmed

# [Add zero address validation in the setPendingGovernance function](https://github.com/code-423n4/2021-10-badgerdao-findings/issues/35) 

# Handle

defsec


# Vulnerability details

## Impact

Since the _pendingGovernance parameter in the setPendingGovernance are used to add governance. In the state variable , proper check up should be done , other wise error in these state variable can lead to redeployment of contract.

## Proof of Concept

1. Navigate to the following contract functions.

"https://github.com/code-423n4/2021-10-badgerdao/blob/9d4734becebd729299f154c0cfa1d3a7f06cccfb/contracts/WrappedIbbtcEth.sol#L50"

"https://github.com/code-423n4/2021-10-badgerdao/blob/9d4734becebd729299f154c0cfa1d3a7f06cccfb/contracts/WrappedIbbtc.sol#L49"

2. Adding zero address into the pending governance leads to failure of governor only functions. 

## Tools Used

Code Review

## Recommended Mitigation Steps

Add proper zero address validation. 

