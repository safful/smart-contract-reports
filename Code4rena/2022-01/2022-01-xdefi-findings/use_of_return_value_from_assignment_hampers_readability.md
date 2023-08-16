## Tags

- bug
- 1 (Low Risk)
- resolved
- sponsor confirmed

# [Use of return value from assignment hampers readability](https://github.com/code-423n4/2022-01-xdefi-findings/issues/2) 

# Handle

TomFrenchBlockchain


# Vulnerability details

## Impact

Reduced readability

## Proof of Concept

In a number of placed we seem to be inlining an assignment with the usage of that variable:

https://github.com/XDeFi-tech/xdefi-distribution/blob/3856a42df295183b40c6eee89307308f196612fe/contracts/XDEFIDistribution.sol#L40

https://github.com/XDeFi-tech/xdefi-distribution/blob/3856a42df295183b40c6eee89307308f196612fe/contracts/XDEFIDistribution.sol#L70

https://github.com/XDeFi-tech/xdefi-distribution/blob/3856a42df295183b40c6eee89307308f196612fe/contracts/XDEFIDistribution.sol#L83

This is quite atypical in my experience and reduces readability: lines which contain require statements and event emission now modify contract storage.

## Recommended Mitigation Steps

Consider whether any small benefits to gas/compactness are worth the reduced clarity.

