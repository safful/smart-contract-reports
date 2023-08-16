## Tags

- bug
- G (Gas Optimization)
- sponsor confirmed
- ATokenYieldSource

# [function _getRefferalCode() can be refactored to a constant variable](https://github.com/code-423n4/2021-06-pooltogether-findings/issues/20) 

# Handle

pauliax


# Vulnerability details

## Impact
function _getRefferalCode() in ATokenYieldSource just returns a constant of uint16(188). To save some gas and improve the readability this can be extracted to a constant variable and used where necessary.

## Recommended Mitigation Steps
 uint16 internal constant REFFERAL_CODE = uint16(188);

