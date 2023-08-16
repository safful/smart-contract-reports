## Tags

- bug
- 0 (Non-critical)
- sponsor confirmed
- resolved

# [contract AaveMarket function setRewards has a misleading revert message](https://github.com/code-423n4/2021-05-88mph-findings/issues/4) 

# Handle

paulius.eth


# Vulnerability details

## Impact
contract AaveMarket function setRewards has a misleading revert message:
   require(newValue.isContract(), "HarvestMarket: not contract");

## Recommended Mitigation Steps
Should be 'AaveMarket', not 'HarvestMarket'.

