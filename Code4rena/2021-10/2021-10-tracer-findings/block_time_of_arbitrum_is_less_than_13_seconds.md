## Tags

- bug
- 0 (Non-critical)
- sponsor confirmed
- disagree with severity

# [BLOCK_TIME of Arbitrum is less than 13 seconds](https://github.com/code-423n4/2021-10-tracer-findings/issues/30) 

# Handle

pauliax


# Vulnerability details

## Impact
It is unclear why the block time is based on ETH mainnet 13s intervals, when in Arbitrum where these contracts are supposed to be deployed block times are faster:
    uint256 public constant BLOCK_TIME = 13; /* in seconds */
I wanted to ask this Tracer's representative on Discord but received no answer so submitting this and you can decide if that was intentional.



