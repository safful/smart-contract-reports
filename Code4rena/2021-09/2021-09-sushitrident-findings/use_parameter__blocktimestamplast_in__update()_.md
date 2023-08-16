## Tags

- bug
- G (Gas Optimization)
- sponsor confirmed

# [Use parameter _blockTimestampLast in _update() ](https://github.com/code-423n4/2021-09-sushitrident-findings/issues/16) 

# Handle

gpersoon


# Vulnerability details

## Impact
The function _update() of ConstantProductPool.sol verifies if blockTimestampLast == 0.
The value of blockTimestampLast is also passed in the parameter _blockTimestampLast  (note: with extra _ )
So _blockTimestampLast could also be used, which saves a bit of gas.

## Proof of Concept
https://github.com/sushiswap/trident/blob/master/contracts/pool/ConstantProductPool.sol#L264
 function _update(...        uint32 _blockTimestampLast) internal {
        ...
        if (blockTimestampLast == 0) {    // could also use _blockTimestampLast

## Tools Used

## Recommended Mitigation Steps
replace
   if (blockTimestampLast == 0) {   
with
   if (_blockTimestampLast == 0) {    

