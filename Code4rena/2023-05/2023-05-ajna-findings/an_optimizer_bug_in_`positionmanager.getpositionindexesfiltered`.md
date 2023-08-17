## Tags

- bug
- 2 (Med Risk)
- satisfactory
- selected for report
- sponsor confirmed
- M-06

# [An Optimizer Bug in `PositionManager.getPositionIndexesFiltered`](https://github.com/code-423n4/2023-05-ajna-findings/issues/306) 

# Lines of code

https://github.com/code-423n4/2023-05-ajna/blob/main/ajna-core/src/PositionManager.sol#L484
https://github.com/code-423n4/2023-05-ajna/blob/main/ajna-core/src/PositionManager.sol#L3


# Vulnerability details


## Impact

There is an optimizer bug in `PositionManager.getPositionIndexesFiltered`.
https://blog.soliditylang.org/2022/06/15/inline-assembly-memory-side-effects-bug/

The Yul optimizer considers all memory writes in the outermost Yul block that are never read from as unused and removes them. The bug is fixed in solidity 0.8.15. But `PositionManager.sol` uses solidity 0.8.14

## Proof of Concept

There is an inline assembly block at the end of `PositionManager.getPositionIndexesFiltered`. The written memory is never read from in the same assembly block. It would trigger the bug to remove the memory write.
https://github.com/code-423n4/2023-05-ajna/blob/main/ajna-core/src/PositionManager.sol#L484
```solidity
    function getPositionIndexesFiltered(
        uint256 tokenId_
    ) external view override returns (uint256[] memory filteredIndexes_) {
        …

        // resize array
        assembly { mstore(filteredIndexes_, filteredIndexesLength) }
    }
```

Unfortunately, `PositionManager` uses solidity 0.8.14 which would suffer from the bug.
https://github.com/code-423n4/2023-05-ajna/blob/main/ajna-core/src/PositionManager.sol#L3
```
pragma solidity 0.8.14;
```


## Tools Used

Manual Review

## Recommended Mitigation Steps

Update the solidity version to 0.8.15



## Assessed type

Other