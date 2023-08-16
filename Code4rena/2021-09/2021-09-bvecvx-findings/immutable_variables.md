## Tags

- bug
- G (Gas Optimization)
- sponsor confirmed

# [Immutable variables](https://github.com/code-423n4/2021-09-bvecvx-findings/issues/26) 

# Handle

pauliax


# Vulnerability details

## Impact
Variables that do not change can be marked as immutable. This greatly reduces gas cots. Examples of such variables are:
  ICvxLocker public LOCKER;
  uint256 MAX_BPS = 10_000;
  address public lpComponent;
  address public reward;


