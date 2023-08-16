## Tags

- bug
- 0 (Non-critical)
- sponsor confirmed

# [veCVXStrategy: Erroneous Comments](https://github.com/code-423n4/2021-09-bvecvx-findings/issues/35) 

# Handle

hickuphh3


# Vulnerability details

### Impact

- L211: `// We receive bCVX -> Convert to bCVX` → `We receive bCVX -> Convert to CVX`
- L443: `/// @notice toLock = 100, lock everything (CVX) you have` → `/// @notice toLock = MAX_BPS, lock everything (CVX) you have` since MAX_BPS (10_000) is the base used

