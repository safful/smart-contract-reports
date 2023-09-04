## Tags

- bug
- 2 (Med Risk)
- primary issue
- selected for report
- sponsor confirmed
- M-07

# [Missing recursive calls handling in `OffchainDNSResolver` CCIP-aware contract](https://github.com/code-423n4/2023-04-ens-findings/issues/124) 

# Lines of code

https://github.com/code-423n4/2023-04-ens/blob/main/contracts/dnsregistrar/OffchainDNSResolver.sol#L109-L113
https://github.com/code-423n4/2023-04-ens/blob/main/contracts/dnsregistrar/OffchainDNSResolver.sol#L119


# Vulnerability details

The `resolveCallback` function from `OffchainDNSResolver` is used as part of the EIP-3668 standard to properly resolve DNS names using an off-chain gateway and validating RRsets against the DNSSEC oracle.


The issue is that the function lacks proper error handling, specifically, a try/catch block to properly bubble up `OffchainLookup` error from the `dnsresolver` extracted from the RRset. As the EIP specifies,

> When a CCIP-aware contract wishes to make a call to another contract, and the possibility exists that the callee may implement CCIP read, the calling contract MUST catch all `OffchainLookup` errors thrown by the callee, and revert with a different error if the `sender` field of the error does not match the callee address.
> [...]
> Where the possibility exists that a callee implements CCIP read, a CCIP-aware contract MUST NOT allow the default solidity behaviour of bubbling up reverts from nested calls. 

## Impact

As per the EIP, the result would be an OffchainLookup that looks valid to the client, as the sender field matches the address of the contract that was called, but does not execute correctly.


## Proof of Concept

1. Client calls `OffchainDNSResolver.resolve`, which reverts with `OffchainLookup`, and prompts the client to execute `resolveCallback` after having fetched the necessary data from the `gatewayURL`
2. The RRset returned by the gateway contains a `dnsresolver` that is a CCIP-aware contract, and also supports the `IExtendedDNSResolver.resolve.selector` interface
3. Calling `IExtendedDNSResolver(dnsresolver).resolve(name,query,context);` could trigger another `OffchainLookup` error, but with a `sender` that does not match the `dnsresolver`, which would be just returned to the client without any modifications
4. As a result, the `sender` field would be incorrect as per the EIP


## Tools Used

Manual review

## Recommended Mitigation Steps

Use the [recommended example](https://eips.ethereum.org/EIPS/eip-3668#example-1) from the EIP in order to support nested lookups. 