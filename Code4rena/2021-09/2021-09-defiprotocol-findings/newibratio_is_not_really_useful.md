## Tags

- bug
- G (Gas Optimization)
- sponsor confirmed

# [newIbRatio is not really useful](https://github.com/code-423n4/2021-09-defiprotocol-findings/issues/215) 

# Handle

pauliax


# Vulnerability details

## Impact
It is unclear why you need this new local variable called newIbRatio if you instantly update and use the storage variable afterwards:
  uint256 newIbRatio = ibRatio * startSupply / totalSupply();
  ibRatio = newIbRatio;

## Recommended Mitigation Steps
  ibRatio = ibRatio * startSupply / totalSupply();

