## Tags

- bug
- 0 (Non-critical)
- sponsor confirmed
- resolved

# [Interface and implementation function declaration differs](https://github.com/code-423n4/2021-04-maple-findings/issues/98) 

# Handle

paulius.eth


# Vulnerability details

## Vulnerability details

ILoan.sol:
function getNextPayment() external view returns (uint256, uint256, uint256, uint256)
Loan.sol:
function getNextPayment() public view returns(uint256, uint256, uint256, uint256, bool)
Such discrepencies appear because implementation contracts do not inherit the interface explicitly (Loan is ILoan), so it does not give compilation errors if the declaration changes.

## Recommended Mitigation Steps

Unify the declarations or even better, make the contract inherit from the interface so you can always be sure that these functions are present.

