## Tags

- bug
- sponsor confirmed
- G (Gas Optimization)
- resolved

# [LibBytes uses itself](https://github.com/code-423n4/2021-10-ambire-findings/issues/58) 

# Handle

pauliax


# Vulnerability details

## Impact
I don't think it's necessary for the library to use itself here:
  library LibBytes {
    using LibBytes for bytes;

## Recommended Mitigation Steps
Remove this 'using' statement as it does not give anything in this case.

