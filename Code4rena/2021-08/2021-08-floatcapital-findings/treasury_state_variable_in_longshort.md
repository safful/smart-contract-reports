## Tags

- bug
- G (Gas Optimization)
- sponsor confirmed
- fixed-in-upstream-repo

# [treasury state variable in LongShort](https://github.com/code-423n4/2021-08-floatcapital-findings/issues/110) 

# Handle

pauliax


# Vulnerability details

## Impact
contract LongShort has a 'treasury' state variable that is not used in any meaningful way. It is only initialized in function initialize and can be changed by function changeTreasury, no other interactions. YieldManagerAave has its own separate treasury variable that it allocates funds to so this dead code can be removed to save some gas at least.


