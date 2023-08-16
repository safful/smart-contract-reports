## Tags

- bug
- 0 (Non-critical)
- sponsor confirmed

# [_getToken not resilient to errors](https://github.com/code-423n4/2022-01-notional-findings/issues/36) 

# Handle

0x1f8b


# Vulnerability details

## Impact
`_getToken` return empty values instead of revert.

## Proof of Concept
The library `TokenHandler` has the `_getToken` method and this method returns an empty struct instead of revert if the currencyId was not found, this can produce in unexpected errors.

## Tools Used
Manual review.

## Recommended Mitigation Steps
revert if it was not found.

