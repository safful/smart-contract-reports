## Tags

- bug
- G (Gas Optimization)
- sponsor confirmed

# ['matured' can be replaced by 'maturityRate' > 0](https://github.com/code-423n4/2021-09-swivel-findings/issues/151) 

# Handle

pauliax


# Vulnerability details

## Impact
boolean flag 'matured' could be removed if you agree to accept maturityRate > 0 as a matured vault, basically replacing:
  require(!matured, 'already matured');
with:
  require(maturityRate == 0, 'already matured');
This would eliminate one storage variable and thus reduce gas usage. The risk is that exchangeRateCurrent can never be 0 as this would mean an immature state.

## Recommended Mitigation Steps
Consider getting rid of 'matured' as per suggestion.


