## Tags

- bug
- 1 (Low Risk)
- sponsor confirmed

# [Swivel: Implement check effect interaction to align with best practices](https://github.com/code-423n4/2021-09-swivel-findings/issues/37) 

# Handle

itsmeSTYJ


# Vulnerability details

## Impact

There is no impact to the funds but to align with [best practices]([https://fravoll.github.io/solidity-patterns/checks_effects_interactions.html](https://fravoll.github.io/solidity-patterns/checks_effects_interactions.html)), it is always better to update internal state before any external function calls.

## Recommended Mitigation Steps

For functions `exitVaultFillingZcTokenExit()` and `exitZcTokenFillingVaultExit()`, you should do `mPlace.custodialExit(...)` to update the internal accounting before transferring the tokens out.

