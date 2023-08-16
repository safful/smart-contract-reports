## Tags

- bug
- G (Gas Optimization)
- sponsor confirmed

# [Pre-calculate known values](https://github.com/code-423n4/2021-10-union-findings/issues/90) 

# Handle

pauliax


# Vulnerability details

## Impact
This duration calculation does not change so can be pre-calculated to reduce gas costs:
  // before
  amount = (vestingAmount * (block.timestamp - lastUpdate)) / (vestingEnd - vestingBegin);

  // after
  uint256 public constant VESTING_DURATION; // constant state variable
  VESTING_DURATION = vestingEnd - vestingBegin; // assign value in the constructor
  amount = (vestingAmount * (block.timestamp - lastUpdate)) / VESTING_DURATION;

Same with this:
  return (token.getPastTotalSupply(blockNumber) * 4e16) / 1e18; //4%


