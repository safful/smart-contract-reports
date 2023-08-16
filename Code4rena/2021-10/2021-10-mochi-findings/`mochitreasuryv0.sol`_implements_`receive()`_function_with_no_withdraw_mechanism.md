## Tags

- bug
- 0 (Non-critical)
- disagree with severity
- sponsor confirmed

# [`MochiTreasuryV0.sol` Implements `receive()` Function With No Withdraw Mechanism](https://github.com/code-423n4/2021-10-mochi-findings/issues/162) 

# Handle

leastwood


# Vulnerability details

## Impact

The `MochiTreasuryV0.sol` contract freely receives ETH from users/other contracts. In the event this does happen, ETH is permanently locked and unrecoverable by the protocol's governance framework.

## Proof of Concept

https://github.com/code-423n4/2021-10-mochi/blob/main/projects/mochi-core/contracts/treasury/MochiTreasuryV0.sol

## Tools Used

Manual code review
Slither

## Recommended Mitigation Steps

Consider enabling ETH withdraws for the governance role.

