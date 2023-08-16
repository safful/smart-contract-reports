## Tags

- bug
- G (Gas Optimization)
- sponsor confirmed
- resolved

# [Gas: Bitmasks creation can be simplified](https://github.com/code-423n4/2021-10-pooltogether-findings/issues/21) 

# Handle

cmichel


# Vulnerability details

In `DrawCalculator._createBitMasks`, the bit masks can just be created by shifting the previous (potentially already shifted) masks by the bit range.
It saves on multiplications and, for me, this is also more intuitive than the current algorithm.

Some pseudocode:
```solidity
function _createBitMasks(IPrizeDistributionBuffer.PrizeDistribution memory _prizeDistribution)
    internal
    pure
    returns (uint256[] memory)
{
    uint256[] memory masks = new uint256[](_prizeDistribution.matchCardinality);
    uint256 _bitRangeMaskValue = (2**_prizeDistribution.bitRangeSize) - 1; // get a decimal representation of bitRangeSize, for example 0xF for bitRangeSize = 4

    if(_prizeDistribution.matchCardinality == 0) return masks;

    masks[0] = _bitRangeMaskValue;
    for (uint8 maskIndex = 1; maskIndex < _prizeDistribution.matchCardinality; maskIndex++) {
        // shift by the "size" of the bit mask each time, 0xF, 0xF0, 0xF00, 0xF000, etc.
        masks[maskIndex] = masks[maskIndex - 1] << _prizeDistribution.bitRangeSize;
    }

    return masks;
}
```

