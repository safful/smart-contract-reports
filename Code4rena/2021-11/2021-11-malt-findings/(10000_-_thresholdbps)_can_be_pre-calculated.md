## Tags

- bug
- G (Gas Optimization)
- sponsor confirmed

# [(10000 - thresholdBps) can be pre-calculated](https://github.com/code-423n4/2021-11-malt-findings/issues/369) 

# Handle

pauliax


# Vulnerability details

## Impact
contract PoolTransferVerification sets thresholdBps but in calculations uses only ```(10000 - thresholdBps)```. Consider pre-calculating to avoid re-evaluation again and again when this function is invoked.

