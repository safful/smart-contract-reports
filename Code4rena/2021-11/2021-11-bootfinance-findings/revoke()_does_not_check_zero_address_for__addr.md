## Tags

- bug
- 1 (Low Risk)
- sponsor confirmed

# [revoke() Does Not Check Zero Address for _addr](https://github.com/code-423n4/2021-11-bootfinance-findings/issues/202) 

# Handle

Meta0xNull


# Vulnerability details

## Impact
revoke() Does Not Check Zero Address for _addr

## Proof of Concept
https://github.com/code-423n4/2021-11-bootfinance/blob/main/vesting/contracts/Vesting.sol#L104-L105

more...

## Tools Used
Manual Review

## Recommended Mitigation Steps
Check _addr for Zero Address

