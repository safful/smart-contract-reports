## Tags

- bug
- 0 (Non-critical)
- sponsor confirmed

# [Used a fixed or pragma that spans only a single `0.x.*`](https://github.com/code-423n4/2021-08-notional-findings/issues/90) 

# Handle

hrkrshnn


# Vulnerability details

## Used a fixed or pragma that spans only a single `0.x.*`

Currently, the pragma `>0.7.0` is used in several contracts. However,
since 0.7.0 and 0.8.0 has breaking changes, especially the safemath by
default, the contracts could be semantically different when compiled via
`0.7.*` and `0.8.*`.


