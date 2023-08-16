## Tags

- bug
- G (Gas Optimization)
- sponsor confirmed

# [Useless initialization to default value](https://github.com/code-423n4/2021-09-sushimiso-findings/issues/122) 

# Handle

pauliax


# Vulnerability details

## Impact
No need for this line in function initMISOMarket as it gets this value by default:
  auctionTemplateId = 0;

## Recommended Mitigation Steps
Consider removing useless initialization.

