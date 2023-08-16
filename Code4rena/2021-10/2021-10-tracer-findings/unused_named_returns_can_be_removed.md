## Tags

- bug
- G (Gas Optimization)
- sponsor confirmed

# [Unused Named Returns Can Be Removed](https://github.com/code-423n4/2021-10-tracer-findings/issues/6) 

# Handle

ye0lde


# Vulnerability details

## Impact

Removing unused named return variables can reduce gas usage and improve code clarity.

## Proof of Concept

The unused named return variables are here.

ChainlinkOracleWrapper.sol:
https://github.com/tracer-protocol/perpetual-pools-contracts/blob/646360b0549962352fe0c3f5b214ff8b5f73ba51/contracts/implementation/ChainlinkOracleWrapper.sol#L57-L67

LeveragedPool.sol
https://github.com/tracer-protocol/perpetual-pools-contracts/blob/646360b0549962352fe0c3f5b214ff8b5f73ba51/contracts/implementation/LeveragedPool.sol#L327-L340
https://github.com/tracer-protocol/perpetual-pools-contracts/blob/646360b0549962352fe0c3f5b214ff8b5f73ba51/contracts/implementation/LeveragedPool.sol#L353-L355

## Tools Used
Visual Studio Code

## Recommended Mitigation Steps
Remove the unused named return variables or use them instead of creating additional variables.


