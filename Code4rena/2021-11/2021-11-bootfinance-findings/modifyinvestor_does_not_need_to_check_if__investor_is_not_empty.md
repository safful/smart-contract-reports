## Tags

- bug
- G (Gas Optimization)
- sponsor confirmed

# [modifyInvestor does not need to check if _investor is not empty](https://github.com/code-423n4/2021-11-bootfinance-findings/issues/281) 

# Handle

pauliax


# Vulnerability details

## Impact
This check in function modifyInvestor is not neccessary:
  require(_investor != address(0), "Invalid old address");

as empty address cannot be added in function addInvestor and later this check will fail:
  require(investors[_investor].amount != 0);

