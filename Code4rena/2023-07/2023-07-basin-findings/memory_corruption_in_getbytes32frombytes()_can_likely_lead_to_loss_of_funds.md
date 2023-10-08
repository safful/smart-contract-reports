## Tags

- bug
- 2 (Med Risk)
- disagree with severity
- downgraded by judge
- judge review requested
- primary issue
- satisfactory
- selected for report
- sponsor acknowledged
- M-01

# [Memory corruption in getBytes32FromBytes() can likely lead to loss of funds](https://github.com/code-423n4/2023-07-basin-findings/issues/263) 

# Lines of code

https://github.com/code-423n4/2023-07-basin/blob/c1b72d4e372a6246e0efbd57b47fb4cbb5d77062/src/libraries/LibBytes.sol#L23


# Vulnerability details

## Description

The `LibBytes` library is used to read and store `uint128` types compactly for Well functions. The function `getBytes32FromBytes()`  will fetch a specific index as `bytes32`.

```solidity
/**
 * @dev Read the `i`th 32-byte chunk from `data`.
 */
function getBytes32FromBytes(bytes memory data, uint256 i) internal pure returns (bytes32 _bytes) {
    uint256 index = i * 32;
    if (index > data.length) {
        _bytes = ZERO_BYTES;
    } else {
        assembly {
            _bytes := mload(add(add(data, index), 32))
        }
    }
}
```

If the index is out of bounds in the data structure, it returns `ZERO_BYTES = bytes32(0)`. The issue is that the OOB check is incorrect. If `index=data.length`, the request is also OOB. For example:
`data.length=0` -> array is empty, `data[0]` is undefined.
`data.length=32` -> array has one `bytes32`, `data[32]` is undefined.

In other words, fetching the last element in the array will return whatever is stored in memory after the `bytes` structure.

## Impact

Users of `getBytes32FromBytes` will receive arbitrary incorrect data. If used to fetch reserves like `readUint128`, this could easily cause severe damage, like incorrect pricing, or wrong logic that leads to loss of user funds.

## POC

The damage is easily demonstrated using the example below:
```solidity

pragma solidity 0.8.17;

contract Demo {
    event Here();
    bytes32 private constant ZERO_BYTES = bytes32(0);


    function corruption_POC(bytes memory data1, bytes memory data2) external returns (bytes32 _bytes) {
        _bytes = getBytes32FromBytes(data1, 1);
    }

    function getBytes32FromBytes(bytes memory data, uint256 i) internal returns (bytes32 _bytes) {
        uint256 index = i * 32;
        if (index > data.length) {
            _bytes = ZERO_BYTES;
        } else {
            emit Here();
            assembly {
                _bytes := mload(add(add(data, index), 32))
            }
        }
    }
}
```

Calling corruption_POC with the following parameters:
```
{
	"bytes data1": "0x2222222222222222222222222222222222222222222222222222222222222222",
	"bytes data2": "0x3333333333333333333333333333333333333333333333333333333333333333"
}
```

The output `bytes32` is:
```
{
	"0": "bytes32: _bytes 0x0000000000000000000000000000000000000000000000000000000000000020"
}
```

The 0x20 value is in fact the size of the `data2` bytes that resides in memory from the call to `getBytes32FromBytes`.

## Tools Used

Manual audit

## Recommended Mitigation Steps

Change the logic to:
```solidity
if (index >= data.length) {
    _bytes = ZERO_BYTES;
} else {
    assembly {
        _bytes := mload(add(add(data, index), 32))
    }
}
```



## Assessed type

Invalid Validation