## Tags

- bug
- 1 (Low Risk)
- sponsor confirmed

# [Sanitize `_weights` in `setWeights` on every use](https://github.com/code-423n4/2021-07-sherlock-findings/issues/115) 

# Handle

cmichel


# Vulnerability details

The `setWeights` function only stores the `uint16` part of `_weights[i]` in storage (`ps.sherXWeight = uint16(_weights[i])`).
However, to calculate `weightAdd/weightSub` the full value (not truncated to 16 bits) is used.
This can lead to discrepancies as the actually added part is different from the one tracked in the `weightAdd` variable.


