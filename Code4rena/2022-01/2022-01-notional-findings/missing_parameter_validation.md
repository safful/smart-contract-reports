## Tags

- bug
- 0 (Non-critical)
- sponsor confirmed

# [Missing parameter validation](https://github.com/code-423n4/2022-01-notional-findings/issues/195) 

# Handle

cmichel


# Vulnerability details

Some parameters of functions are not checked for invalid values:
- `TreasuryManager.setPriceOracle: oracleAddress`: could break things
- `TreasuryManager.setSlippageLimit: slippageLimit`: should be `<= SLIPPAGE_LIMIT_PRECISION`

## Impact
Wrong user input or wallets defaulting to the zero addresses for a missing input can lead to the contract needing to redeploy or wasted gas.

## Recommended Mitigation Steps
Validate the parameters.

