## Tags

- bug
- G (Gas Optimization)
- sponsor confirmed

# [Gas saving in ngmi(uint256,uint256,bytes32[])](https://github.com/code-423n4/2021-11-fei-findings/issues/13) 

# Handle

tqts


# Vulnerability details

## Impact
Saving of one SLOAD instruction

## Proof of Concept
The value of `claimed[thisSender] + multiplier` is used twice in `ngmi(uint256,uint256,bytes32[])`. This involves a SLOAD each time it is calculated.
Usages in lines 73 and 76 of TRIBERagequit.sol.

## Tools Used
Manual review

## Recommended Mitigation Steps
Precache the value of the expression right before line 72. Worst case, if the require fails, no extra gas is used. Best case, if the require succeeds, the SLOAD in line 76 is saved.

