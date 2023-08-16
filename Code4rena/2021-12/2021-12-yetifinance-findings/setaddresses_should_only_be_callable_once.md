## Tags

- bug
- 1 (Low Risk)
- disagree with severity
- sponsor confirmed

# [setAddresses should only be callable once](https://github.com/code-423n4/2021-12-yetifinance-findings/issues/240) 

# Handle

pauliax


# Vulnerability details

## Impact
function setAddresses in contract Whitelist is intended to be invoked only once (confirmed with the sponsor) but currently, it has no prevention from being called multiple times.

Maybe this should also be prevented in sYETIToken's setAddresses and ThreePieceWiseLinearPriceCurve's setAddresses.

## Recommended Mitigation Steps
Prevent repeated access of setAddresses in Whitelist and potentially in sYETIToken and ThreePieceWiseLinearPriceCurve.

