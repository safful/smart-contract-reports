## Tags

- bug
- 1 (Low Risk)
- sponsor confirmed

# [LibMath sumN can iterate over array](https://github.com/code-423n4/2021-06-tracer-findings/issues/89) 

# Handle

cmichel


# Vulnerability details

When `LibMath.sumN` function does not check if `n <= arr.length` and can therefore fail if called with `n > arr.length`.

## Impact
The caller must always check that it's called with an argument that is less than `n` which is inconvenient.

## Recommendation
Change the condition to iterate up to `min(n, arr.length)`.


