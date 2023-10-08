## Tags

- bug
- 2 (Med Risk)
- disagree with severity
- downgraded by judge
- primary issue
- satisfactory
- selected for report
- sponsor confirmed
- M-02

# [Due to slot confusion, reserve amounts in the pump will be corrupted, resulting in wrong oracle values](https://github.com/code-423n4/2023-07-basin-findings/issues/260) 

# Lines of code

https://github.com/code-423n4/2023-07-basin/blob/c1b72d4e372a6246e0efbd57b47fb4cbb5d77062/src/libraries/LibBytes16.sol#L45
https://github.com/code-423n4/2023-07-basin/blob/c1b72d4e372a6246e0efbd57b47fb4cbb5d77062/src/libraries/LibLastReserveBytes.sol#L58


# Vulnerability details

## Description

The MultiFlowPump contract stores reserve counts on every update, using the libraries LibBytes16 and LibLastReserveBytes. Those libs pack `bytes16` values efficiently with the `storeBytes16()` and `storeLastReserves` functions.
In case of an odd number of items, the last storage slot will be half full. Care must be taken to not step over the previous value in that slot. This is done correctly in `LibBytes`:
```solidity
if (reserves.length & 1 == 1) {
    require(reserves[reserves.length - 1] <= type(uint128).max, "ByteStorage: too large");
    iByte = maxI * 64;
    assembly {
        sstore(
            // @audit - overwrite SLOT+MAXI*32
            add(slot, mul(maxI, 32)),
                                                       // @audit - read SLOT+MAXI*32 (2nd entry)
            add(mload(add(reserves, add(iByte, 32))), shl(128, shr(128, sload(add(slot, mul(maxI, 32))))))
        )
    }
}
```

As can be seen, it overwrites the slot with the previous 128 bits in the upper half of the slot, only setting the lower 128 bytes.

However, the wrong slot is read in the other two libraries. For example, in `storeLastReserves()`:
```solidity
if (reserves.length & 1 == 1) {
    iByte = maxI * 64;
    assembly {
        sstore(
            // @audit - overwrite SLOT+MAXI*32
            add(slot, mul(maxI, 32)),
                                                      // @audit - read SLOT+MAXI (2nd entry)??
            add(mload(add(reserves, add(iByte, 32))), shr(128, shl(128, sload(add(slot, maxI)))))
        )
    }
}
```

The error is not multiplying `maxI` before adding it to `slot`. This means that the reserves count encoded in lower 16 bytes in `add(slot, mul(maxI, 32))` will have the value of a reserve in a much lower index. 
Slots are used in 32 byte increments, i.e. S, S+32, S+64...
When `maxI==0`, the intended slot and the actual slot overlap. When `maxI` is 1..31, the read slot happens to be zero (unused), so the first actual corruption occurs on `maxI==32`. By substitution, we get:
`SLOT[32*32] = correct reserve | SLOT[32]`
In other words, the 4rd reserve (stored in lower 128 bits of  `SLOT[32]`) will be written to the 64th reserve. 

The Basin pump is intended to support an arbitrary number of reserves safely, therefore the described storage corruption impact is in scope.

## Impact

Reserve amounts in the pump will be corrupted, resulting in wrong oracle values.

## POC

1. A reserve update is triggered on the pump when some Well action occurs.
2. Suppose reserves are array `[0,1,2,...,63,64]`
3. Reserve count is odd, so affected code block is reached
4. `SLOT[32*32] = UPPER: 64 | LOWER: SLOT[32] = 64 | 3`


## Tools Used

Manual audit

## Recommended Mitigation Steps

Change the `sload()` operation in both affected functions to `sload(add(slot, mul(maxI, 32)`


## Assessed type

en/de-code