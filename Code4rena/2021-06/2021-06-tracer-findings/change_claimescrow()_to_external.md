## Tags

- bug
- 0 (Non-critical)
- sponsor confirmed

# [Change claimEscrow() to external](https://github.com/code-423n4/2021-06-tracer-findings/issues/128) 

# Handle

0xsanson


# Vulnerability details

## Impact
The claimEscrow(...) function in Liquidation.sol can be set external instead of public since it's not used in the contract. (code clarity and gas savings)

## Proof of Concept
https://github.com/code-423n4/2021-06-tracer/blob/main/src/contracts/Liquidation.sol#L109

## Tools Used
Manual analysis

## Recommended Mitigation Steps
Change public to external.

