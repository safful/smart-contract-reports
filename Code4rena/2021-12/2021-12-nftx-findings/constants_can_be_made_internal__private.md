## Tags

- bug
- G (Gas Optimization)
- resolved
- sponsor confirmed

# [Constants can be made internal / private](https://github.com/code-423n4/2021-12-nftx-findings/issues/209) 

# Handle

Dravee


# Vulnerability details

## Impact
Since the defined constants are unneeded elsewhere, it can be defined to be internal or private to save gas.

## Proof of Concept
```
https://github.com/code-423n4/2021-12-nftx/blob/main/nftx-protocol-v2/contracts/solidity/NFTXInventoryStaking.sol#L28-L30
```

## Tools Used
VS Code

## Recommended Mitigation Steps
Change the visibility from public to private or internal

