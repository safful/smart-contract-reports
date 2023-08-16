## Tags

- bug
- sponsor confirmed
- 0 (Non-critical)
- disagree with severity
- resolved

# [Unnecessary imports](https://github.com/code-423n4/2021-10-pooltogether-findings/issues/50) 

# Handle

pauliax


# Vulnerability details

## Impact
There are imports that are not necessary. They will increase the size of deployment with no real benefit. Consider removing them to save some gas. 
Examples of such imports are:
in contract DrawBuffer imported twice:
import "./interfaces/IDrawBeacon.sol";

in contract DrawCalculator:
import "./libraries/DrawRingBufferLib.sol";
import "./PrizeDistributor.sol";

in contract PrizeDistributor:
import "./interfaces/IDrawBeacon.sol";

## Recommended Mitigation Steps
Consider removing imports that are not actually used in the contract.

