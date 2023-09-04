## Tags

- bug
- 2 (Med Risk)
- primary issue
- selected for report
- sponsor disputed
- M-03

# [Offchain name resolution would fail despite the located DNS resolver being fully functional](https://github.com/code-423n4/2023-04-ens-findings/issues/256) 

# Lines of code

https://github.com/code-423n4/2023-04-ens/blob/45ea10bacb2a398e14d711fe28d1738271cd7640/contracts/dnsregistrar/OffchainDNSResolver.sol#L104


# Vulnerability details

## Description

In OffchainDNSResolver, `resolveCallback` parses resource  records received off-chain and extracts the DNS resolver address:
```
		// Look for a valid ENS-DNS TXT record
		(address dnsresolver, bytes memory context) = parseRR(
			iter.data,
			iter.rdataOffset,
			iter.nextOffset
		);
```

The contract supports three methods of resolution through the resolver:
1. IExtendedDNSResolver.resolve
2. IExtendedResolver.resolve
3. Arbitrary call with the `query` parameter originating in `resolve()`

Code is below:
```
            // If we found a valid record, try to resolve it
            if (dnsresolver != address(0)) {
                if (
                    IERC165(dnsresolver).supportsInterface(
                        IExtendedDNSResolver.resolve.selector
                    )
                ) {
                    return
                        IExtendedDNSResolver(dnsresolver).resolve(
                            name,
                            query,
                            context
                        );
                } else if (
                    IERC165(dnsresolver).supportsInterface(
                        IExtendedResolver.resolve.selector
                    )
                ) {
                    return IExtendedResolver(dnsresolver).resolve(name, query);
                } else {
                    (bool ok, bytes memory ret) = address(dnsresolver)
                        .staticcall(query);
                    if (ok) {
                        return ret;
                    } else {
                        revert CouldNotResolve(name);
                    }
                }
            }
```

The issue is that a resolver could support option (3), but execution would revert prior to the `query` call. The function uses `supportsInterface` in an unsafe way. It should first check that the contract implements ERC165, which will guarantee the call won't revert. Dynamic resolvers are not likely in practice to implement ERC165 as there's no specific signature to support ahead of time.

## Impact

Resolution with custom DNS resolvers are likely to fail.

## Tools Used

Manual audit

## Recommended Mitigation Steps

Use the OZ `ERC165Checker.sol` library, which [addresses](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/f959d7e4e6ee0b022b41e5b644c79369869d8411/contracts/utils/introspection/ERC165Checker.sol#L36) the issue:
```
    function supportsInterface(address account, bytes4 interfaceId) internal view returns (bool) {
        // query support of both ERC165 as per the spec and support of _interfaceId
        return supportsERC165(account) && supportsERC165InterfaceUnchecked(account, interfaceId);
    }
```