## Tags

- bug
- G (Gas Optimization)
- sponsor confirmed

# [Gas optimization](https://github.com/code-423n4/2021-12-sublime-findings/issues/34) 

# Handle

0x1f8b


# Vulnerability details

## Impact
Gas saving.

## Proof of Concept
The method initializeRepayment inside the contract Repayments has multipe storage access, it's better to get a pointer of the `RepaymentConstants` with the `storage` keyword in order to avoid seeking and storage access.

## Tools Used
Manual review

## Recommended Mitigation Steps
Use storage keyword in order to save gas

