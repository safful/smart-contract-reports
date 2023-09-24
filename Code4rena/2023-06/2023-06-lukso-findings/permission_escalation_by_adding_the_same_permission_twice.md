## Tags

- bug
- 2 (Med Risk)
- disagree with severity
- downgraded by judge
- high quality report
- primary issue
- satisfactory
- selected for report
- M-08

# [Permission escalation by adding the same permission twice](https://github.com/code-423n4/2023-06-lukso-findings/issues/13) 

# Lines of code

https://github.com/lukso-network/lsp-smart-contracts/blob/v0.10.2/contracts/LSP6KeyManager/LSP6Utils.sol#L169-L177


# Vulnerability details

## Impact
The function `combinePermissions` adds permission to availabe permissions.
If the same permission is added twice, then this will result in a new and different permission.
For example adding `_PERMISSION_STATICCALL` twice results in `_PERMISSION_SUPER_DELEGATECALL`.

This way accidentally dangerous permissions can be set.
Once someone has a dangerous permission, for example `_PERMISSION_SUPER_DELEGATECALL`, they can change all storage parameters of the Universion Profile and then steal all the assets.

## Proof of Concept
The following code shows this. Run with `forge test -vv` to see the console output.

```solidity
pragma solidity ^0.8.13;

import "../../contracts/LSP6KeyManager/LSP6KeyManager.sol";
import "../../contracts/LSP0ERC725Account/LSP0ERC725Account.sol";
import "../../contracts/LSP2ERC725YJSONSchema/LSP2Utils.sol";
import "../../contracts/LSP6KeyManager/LSP6Constants.sol";
import "./UniversalProfileTestsHelper.sol";

contract SetDataRestrictedController is UniversalProfileTestsHelper {
    LSP0ERC725Account public mainUniversalProfile;
    LSP6KeyManager public keyManagerMainUP;
    address public mainUniversalProfileOwner;
    address public combineController;

    function setUp() public {
        mainUniversalProfileOwner = vm.addr(1);
        vm.label(mainUniversalProfileOwner, "mainUniversalProfileOwner");
        combineController = vm.addr(10);
        vm.label(combineController, "combineController");
        mainUniversalProfile = new LSP0ERC725Account(mainUniversalProfileOwner);

        // deploy LSP6KeyManagers
        keyManagerMainUP = new LSP6KeyManager(address(mainUniversalProfile));
        transferOwnership(
            mainUniversalProfile,
            mainUniversalProfileOwner,
            address(keyManagerMainUP)
        );        
    }
    
    function testCombinePermissions() public {       
        bytes32[] memory ownerPermissions = new bytes32[](3);
        ownerPermissions[0] = _PERMISSION_STATICCALL;
        ownerPermissions[1] = _PERMISSION_STATICCALL;
        givePermissionsToController(
            mainUniversalProfile,
            combineController,
            address(keyManagerMainUP),
            ownerPermissions
        );        
        bytes32 key = LSP2Utils.generateMappingWithGroupingKey(_LSP6KEY_ADDRESSPERMISSIONS_PERMISSIONS_PREFIX,bytes20(combineController));
        bytes memory r = mainUniversalProfile.getData(key);
        console.logBytes(r); // 0x00..4000  SUPER_DELEGATECALL
    }
}
```
See [LSP6Utils.sol#L169-L177](https://github.com/lukso-network/lsp-smart-contracts/blob/v0.10.2/contracts/LSP6KeyManager/LSP6Utils.sol#L169-L177) for the code of `combinePermissions`.

## Tools Used
Foundry

## Recommended Mitigation Steps
The permissions shouldn't be added, but they should be `OR`d.
Here is a way to solve this:
```diff
function combinePermissions(bytes32[] memory permissions) internal pure returns (bytes32) {
    uint256 result = 0;
    for (uint256 i = 0; i < permissions.length; i++) {
-      result += uint256(permissions[i]);
+      result |= uint256(permissions[i]);
    }
    return bytes32(result);
}
```



## Assessed type

Math