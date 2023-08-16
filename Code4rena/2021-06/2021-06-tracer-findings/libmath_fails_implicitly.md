## Tags

- bug
- 1 (Low Risk)
- sponsor confirmed

# [LibMath fails implicitly](https://github.com/code-423n4/2021-06-tracer-findings/issues/88) 

# Handle

cmichel


# Vulnerability details


When `LibMath.abs` is called with -2^255 (`type(int256).min`), it tries to multiply it by `-1` but it'll fail as it exceeds the max signed 256-bit integers.

## Impact
The function will fail with an implicit error that might be hard to locate.

## Recommendation
Throw an error similar to `toInt256` like `int256 overflow`.


