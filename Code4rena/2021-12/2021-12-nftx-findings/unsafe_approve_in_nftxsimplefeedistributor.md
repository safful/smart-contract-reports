## Tags

- bug
- 1 (Low Risk)
- resolved
- sponsor confirmed

# [Unsafe approve in NFTXSimpleFeeDistributor](https://github.com/code-423n4/2021-12-nftx-findings/issues/186) 

# Handle

0x1f8b


# Vulnerability details

## Impact
Unsafe approve was done.

## Proof of Concept
In the method `NFTXSimpleFeeDistributor._sendForReceiver` it's made a approve without checking the boolean result, ERC20 standard specify that the token can return false if the approve was not made, so it's mandatory to check the result of approve methods.

## Tools Used
Manual review

## Recommended Mitigation Steps
Use safe approve or check the boolean result

