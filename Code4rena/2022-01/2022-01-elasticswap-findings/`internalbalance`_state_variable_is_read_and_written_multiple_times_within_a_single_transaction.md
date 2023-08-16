## Tags

- bug
- G (Gas Optimization)
- sponsor confirmed

# [`internalBalance` state variable is read and written multiple times within a single transaction](https://github.com/code-423n4/2022-01-elasticswap-findings/issues/55) 

# Handle

Ruhum


# Vulnerability details

## Impact
The `internalBalances` state variable is used extensively throughout the `Exchange` contract. Reading and writing to storage is expensive. Instead of working the state variable directly, the functions should work with a  cached memory variable. The final value should then be saved to storage.

## Proof of Concept
There are too many places where this is happening. Most prominently in the `MathLib` library, where the state variable is passed around as a function parameter. Working with a cached version will be way cheaper. 

## Tools Used

## Recommended Mitigation Steps
Replace the storage variable with a cached memory variable. The library has to be refactored to return the modified values so they can be written back to storage.

