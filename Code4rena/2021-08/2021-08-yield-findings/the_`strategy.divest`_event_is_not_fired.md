## Tags

- bug
- 1 (Low Risk)
- sponsor confirmed
- Strategy

# [The `Strategy.Divest` event is not fired](https://github.com/code-423n4/2021-08-yield-findings/issues/25) 

# Handle

cmichel


# Vulnerability details

The `Strategy.Divest` event is not fired.

## Impact
Off-chain scripts that rely on this event won't work.

## Recommended Mitigation Steps
Emit this event in `endPool`.

