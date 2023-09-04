## Tags

- bug
- 2 (Med Risk)
- disagree with severity
- primary issue
- selected for report
- sponsor confirmed
- M-05

# [Unintentionally register a non-relevant DSN name owner](https://github.com/code-423n4/2023-04-ens-findings/issues/198) 

# Lines of code

https://github.com/code-423n4/2023-04-ens/blob/45ea10bacb2a398e14d711fe28d1738271cd7640/contracts/dnsregistrar/DNSRegistrar.sol#L158


# Vulnerability details

## Impact
If a user proves and claims a DNS name using a wrong address format, it executes successfully without getting any error and the DNS name owner will be changed to a new unknown address.
I considered this as medium severity, as it is high impact finding with low likelihood. Cause the person who owns the new address can take control of the ENS name and transfer its ownership to another account. But because if a person finds out, she can immediately replace the correct address, the probability of such an event is low. 

## Proof of Concept
In the following scenario, I provided a value called `arbitrarybytes` which is 22 bytes and set it as the 'foo.test' DNS owner address. `proveAndClaim()` function will execute successfully. Finally, the owner of the ENS name would be the value set as `newOwner` which is the last 20 bytes (from the right) of the provided value in `arbitrarybytes`. 

```js
    const arbitrarybytes= '0x9fD6E51AaD88f6f4CE6aB8827279CFFFb92266332265'
    const newOwner= '0xe51Aad88f6F4CE6aB8827279cFFFB92266332265'
    const proof = [
      hexEncodeSignedSet(rootKeys(expiration, inception)),
      hexEncodeSignedSet(testRrset('foo.test', arbitrarybytes)),
    ]

    await registrar.proveAndClaim(utils.hexEncodeName('foo.test'), proof, {
      from: accounts[0],
    })

    assert.equal(await ens.owner(namehash.hash('foo.test')), newOwner)
```
## Tools Used
vscode
## Recommended Mitigation Steps
To check the validity of the owner address, the code first checks for the prefix which must be `a=0x`, and second for the length of the address which should not be less than 20 bytes or 40 characters through `hexToAddress()` function. The length of the address can also be checked to not be larger than 40 characters.
```solidity
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