## Tags

- bug
- 2 (Med Risk)
- primary issue
- selected for report
- sponsor acknowledged
- M-19

# [CLOCK_MODE() will not work properly for Arbitrum or Optimism due to block.number](https://github.com/code-423n4/2023-06-lybra-findings/issues/114) 

# Lines of code

https://github.com/code-423n4/2023-06-lybra/blob/main/contracts/lybra/governance/LybraGovernance.sol#L152


# Vulnerability details

## Impact
CLOCK_MODE will not work correctly on Arbitrum.

## Proof of Concept
According to Arbitrum (Docs)[https://developer.offchainlabs.com/time] block.number returns the most recently synced L1 block number. Once per minute the block number in the Sequencer is synced to the actual L1 block number.Using block.number as a clock can lead to inaccurate timing.

It also presents an issue for (Optimism)[https://community.optimism.io/docs/developers/build/differences/#block-numbers-and-timestamps] because each transaction is it's own block.

https://github.com/code-423n4/2023-06-lybra/blob/main/contracts/lybra/governance/LybraGovernance.sol#L152

## Tools Used

## Recommended Mitigation Steps
Use block.timestamp rather than block.number


## Assessed type

Timing