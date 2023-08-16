## Tags

- bug
- sponsor confirmed
- G (Gas Optimization)
- resolved

# [Duplicate math operations](https://github.com/code-423n4/2021-10-ambire-findings/issues/57) 

# Handle

pauliax


# Vulnerability details

## Impact
First perform the addition and only then check the length to avoid this duplicate math operation:
    require(b.length >= index + 32, "BytesLib: length");
    // Arrays are prefixed by a 256 bit length parameter
    index += 32;
Or if you want to stay with this approach, then at least consider using the 'unchecked' keyword when this addition is performed the second time as then ready know this can't overflow. Also, in function recoverAddrImpl the same operation is performed twice:
  sig.length - 33

## Recommended Mitigation Steps
Refactor duplicate math operations.

