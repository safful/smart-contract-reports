## Tags

- bug
- 2 (Med Risk)
- disagree with severity
- downgraded by judge
- primary issue
- satisfactory
- selected for report
- sponsor confirmed
- M-03

# [Due to bit-shifting errors, reserve amounts in the pump will be corrupted, resulting in wrong oracle values](https://github.com/code-423n4/2023-07-basin-findings/issues/259) 

# Lines of code

https://github.com/code-423n4/2023-07-basin/blob/c1b72d4e372a6246e0efbd57b47fb4cbb5d77062/src/libraries/LibBytes16.sol#L45


# Vulnerability details

## Description

It is advised to first read finding: `Due to slot confusion, reserve amounts in the pump will be corrupted, resulting in wrong oracle values`, which provides all the contextual information for this separate bug.

We've discussed how a wrong `sload()` source slot leads to corruption of the reserves. In `LibBytes16`, another confusion occurs.
Recall the correct storage overwriting done in `LibBytes`:
```solidity
assembly {
    sstore(
        // @audit - overwrite SLOT+MAXI*32
        add(slot, mul(maxI, 32)),
                                                   // @audit - read SLOT+MAXI*32 (2nd entry)
        add(mload(add(reserves, add(iByte, 32))), shl(128, shr(128, sload(add(slot, mul(maxI, 32))))))
    )
}
```

Importantly, it **clears** the lower 128 bytes of the source slot and replaces the upper 128 bytes of the dest slot using the upper 128 bytes of  the source slot:
`shl(128,shr(128,SOURCE))`

In `storeBytes16()`, the `shl()` operation has been discarded. This means the code will use the upper 128 bytes of SOURCE to overwrite the lower 128 bytes in DEST.
```
if (reserves.length & 1 == 1) {
    iByte = maxI * 64;
    assembly {
        sstore(
        // @audit - overwrite SLOT+MAXI*32
            add(slot, mul(maxI, 32)),
                                                    // @audit - overwrite lower 16 bytes with upper 16 bytes?
            add(mload(add(reserves, add(iByte, 32))), shr(128, sload(add(slot, maxI))))
        )
    }
}
```

In other words, regardless of the SLOT being read, instead of keeping the lower 128 bits as is, it stores whatever happens to be in the upper 128 bits. Note this is a **completely** different error from the slot confusion, which happens to be in the same line of code.

## Impact

Reserve amounts in the pump will be corrupted, resulting in wrong oracle values

## POC

Assume slot confusion bug has been corrected for clarity.

1. A reserve update is triggered on the pump when some Well action occurs.
2. Suppose reserves are array `[0,1,2,...,63,64]`
3. Suppose previous reserves are array `[P0,P1,...,P64]`
4. Reserve count is odd, so affected code block is reached
5. `SLOT[32*32] = UPPER: 64 | LOWER: UPPER(SLOT[32*32]) = 64 | P64`


## Tools Used

Manual audit

## Recommended Mitigation Steps

Replace the affected line with the calculation below:
`shr(128, shl(128, SLOT))`
This will use the lower 128 bytes and clear the upper 128 bytes,as intended.


## Assessed type

en/de-code