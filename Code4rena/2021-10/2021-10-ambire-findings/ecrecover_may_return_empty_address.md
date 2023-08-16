## Tags

- bug
- sponsor confirmed
- 1 (Low Risk)
- resolved

# [ecrecover may return empty address](https://github.com/code-423n4/2021-10-ambire-findings/issues/56) 

# Handle

pauliax


# Vulnerability details

## Impact
There is a common issue that ecrecover returns empty (0x0) address when the signature is invalid. function recoverAddrImpl should check that before returning the result of ecrecover.

## Recommended Mitigation Steps
See the solution here: https://github.com/OpenZeppelin/openzeppelin-contracts/blob/v3.4.0/contracts/cryptography/ECDSA.sol#L68

