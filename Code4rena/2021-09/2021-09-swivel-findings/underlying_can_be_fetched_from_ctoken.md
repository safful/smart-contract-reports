## Tags

- bug
- 0 (Non-critical)
- sponsor confirmed
- disagree with severity

# [Underlying can be fetched from cToken](https://github.com/code-423n4/2021-09-swivel-findings/issues/142) 

# Handle

pauliax


# Vulnerability details

## Impact
When creating the market (function createMarket), you do not need to specify the address of underlying, it would be less error-prone to dynamically get this from cToken: https://github.com/compound-finance/compound-protocol/blob/master/contracts/CTokenInterfaces.sol#L254 

## Recommended Mitigation Steps
While only the admin can create new markets, I think it would still be nice to algorithmically ensure that this underlying token belongs to this cToken and do not leave a chance for human errors.

