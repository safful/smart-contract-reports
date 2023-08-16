## Tags

- bug
- 1 (Low Risk)
- disagree with severity
- sponsor confirmed

# [Function documentation incorrect for `ConcatStrings::appendUint`](https://github.com/code-423n4/2021-10-slingshot-findings/issues/47) 

# Handle

pmerkleplant


# Vulnerability details

The documentation for the function `appendUint` in `ConcatStrings.sol` is incorrect.
It states: "Concat two strings". However, the function concats a string and a
uint256.

See: [line 19 in ConcatStrings.sol](https://github.com/code-423n4/2021-10-slingshot/blob/main/contracts/lib/ConcatStrings.sol#L19)

