## Tags

- bug
- 2 (Med Risk)
- satisfactory
- selected for report
- sponsor disputed
- M-02

# [Invalid addresses will be accepted as resolvers, possibly bricking assets](https://github.com/code-423n4/2023-04-ens-findings/issues/258) 

# Lines of code

https://github.com/code-423n4/2023-04-ens/blob/45ea10bacb2a398e14d711fe28d1738271cd7640/contracts/utils/HexUtils.sol#L75


# Vulnerability details

## Description

The `hexToAddress` utility parses a string into an address type.
```
    function hexToAddress(
        bytes memory str,
        uint256 idx,
        uint256 lastIdx
    ) internal pure returns (address, bool) {
        if (lastIdx - idx < 40) return (address(0x0), false);
        (bytes32 r, bool valid) = hexStringToBytes32(str, idx, lastIdx);
        return (address(uint160(uint256(r))), valid);
    }
```

The issue is that in the return statement, the `bytes32` type is reduced from 256 bits to 160 bits, without checking that the truncated bytes are zero. When the upper bits are not zero, the `bytes32` is not a valid address and the function must return `false` in the second parameter. However, it instead returns an incorrect trimmed number.

The utility is used in the flow below:
```
proveAndClaim
	_claim
		getOwnerAddress
			parseRR
				parseString
					hexToAdress
```

The incorrect address would be returned from `_claim` and then used as the owner address for `ens.setSubnodeOwner`. This would brick the node.

```
function proveAndClaim(
    bytes memory name,
    DNSSEC.RRSetWithSignature[] memory input
) public override {
    (bytes32 rootNode, bytes32 labelHash, address addr) = _claim(
        name,
        input
    );
    ens.setSubnodeOwner(rootNode, labelHash, addr);
}
```

## Impact

Lack of validation can lead to ENS addresses becoming permanently bricked.

## Tools Used

Manual audit

## Recommended Mitigation Steps

Add a check for the upper 96 bits in the highlighted return statement.

## Note for judge 

Historically, the judge has awarded medium severity to various issues which rely on some user error, are easy to check/fix and present material risk. I respect this line of thought and for the sake of consistency I believe this submission should be judged similarly.