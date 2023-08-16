## Tags

- bug
- 0 (Non-critical)
- disagree with severity
- sponsor confirmed

# [`GatewayVault` events not used](https://github.com/code-423n4/2021-12-mellow-findings/issues/45) 

# Handle

cmichel


# Vulnerability details

The `CollectProtocolFees` and `CollectStrategyFees` events in `GatewayVault` are not used.

## Impact
Unused code can hint at programming or architectural errors.

## Recommended Mitigation Steps
Use it or remove it.

