## Tags

- bug
- G (Gas Optimization)
- sponsor confirmed

# [Itteration over all the timelocks when revoking the user](https://github.com/code-423n4/2021-11-bootfinance-findings/issues/285) 

# Handle

pauliax


# Vulnerability details

## Impact
When revoking the user, there is no need to iterrate over all his timelocks again and calculate the total amount as it should already be stored in a benTotal[_addr] mapping:
  uint256 locked = 0;
  for (uint256 i = 0; i < timelocks[_addr].length; i++) {
      locked = locked.add(timelocks[_addr][i].amount);
  }


