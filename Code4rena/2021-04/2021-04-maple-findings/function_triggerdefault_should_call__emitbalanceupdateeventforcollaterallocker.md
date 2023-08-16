## Tags

- bug
- 0 (Non-critical)
- sponsor confirmed
- resolved

# [Function triggerDefault should call _emitBalanceUpdateEventForCollateralLocker](https://github.com/code-423n4/2021-04-maple-findings/issues/99) 

# Handle

paulius.eth


# Vulnerability details

## Vulnerability details

function triggerDefault should call _emitBalanceUpdateEventForCollateralLocker to emit event as the balance of CollateralLocker changes after calling LoanLib.liquidateCollateral.


