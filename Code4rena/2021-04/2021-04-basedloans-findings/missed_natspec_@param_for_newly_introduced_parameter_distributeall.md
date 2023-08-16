## Tags

- bug
- 0 (Non-critical)
- sponsor confirmed

# [Missed NatSpec @param for newly introduced parameter distributeAll](https://github.com/code-423n4/2021-04-basedloans-findings/issues/22) 

# Handle

0xRajeev


# Vulnerability details

## Impact

The distributeSupplierComp() function has been modified to take in a third parameter which is a boolean distributeAll. But the corresponding NatSpec comments for the function have not been updated to add this new parameter. This could lead to minor confusion where NatSpec is consulted.

## Proof of Concept

https://github.com/code-423n4/2021-04-basedloans/blob/5c8bb51a3fdc334ea0a68fd069be092123212020/code/contracts/Comptroller.sol#L1238-L1243


## Tools Used

Manual Analysis

## Recommended Mitigation Steps

Add @param for distributeAll parameter.

