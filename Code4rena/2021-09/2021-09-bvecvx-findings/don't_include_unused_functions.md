## Tags

- bug
- 1 (Low Risk)
- sponsor confirmed

# [Don't include unused functions](https://github.com/code-423n4/2021-09-bvecvx-findings/issues/4) 

# Handle

tensors


# Vulnerability details

## Impact
The code includes unused functions, like tend(), L319. It's best practice to remove these. It will also save gas.

## Proof of Concept
https://github.com/code-423n4/2021-09-bvecvx/blob/1d64bd58c7a4224cc330cef283561e90ae6a3cf5/veCVX/contracts/veCVXStrategy.sol#L319

## Recommended Mitigation Steps
Remove the unused function.

