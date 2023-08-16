## Tags

- bug
- 1 (Low Risk)
- sponsor confirmed

# [Open TODOs in `ERC1155Action`](https://github.com/code-423n4/2021-08-notional-findings/issues/63) 

# Handle

cmichel


# Vulnerability details

## Vulnerability Details
The `ERC1155Action._checkPostTransferEvent` has open TODOs:

```solidity
// TODO: retrieve revert string
require(status, "Call failed");
```

## Impact
Open TODOs can hint at programming or architectural errors that still need to be fixed.

## Recommended Mitigation Steps
Resolve the TODO and bubble up the error.

