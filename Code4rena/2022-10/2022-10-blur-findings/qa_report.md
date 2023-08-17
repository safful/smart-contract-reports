## Tags

- bug
- QA (Quality Assurance)
- selected for report

# [QA Report](https://github.com/code-423n4/2022-10-blur-findings/issues/284) 

# Low

## Remove `pragma abicoder v2` 
You are using solidity v0.8 since that version and above the ABIEncoderV2 is not experimental anymore - it is actually enabled by default by the compiler.
Remove lines;
[BlurExchange.sol#L3](https://github.com/code-423n4/2022-10-blur/blob/main/contracts/BlurExchange.sol#L3)
[ExecutionDelegate.sol#L3](https://github.com/code-423n4/2022-10-blur/blob/main/contracts/ExecutionDelegate.sol#L3)
[interfaces/IBlurExchange.sol#L3](https://github.com/code-423n4/2022-10-blur/blob/main/contracts/interfaces/IBlurExchange.sol#L3)
[test/TestBlurExchange.sol#L3](https://github.com/code-423n4/2022-10-blur/blob/main/contracts/test/TestBlurExchange.sol#L3)

## Use latest open zeppelin contracts
Your current version of `@openzeppelin/contracts` is 4.4.1 and latest version is 4.7.3
Your current version of `@openzeppelin/contracts-upgradeable` is ^4.6.0 and latest version is ^4.7.3

## Use named imports instead of plain `import 'file.sol'
You use regular imports on lines:
[BlurExchange.sol/#L5-L16](https://github.com/code-423n4/2022-10-blur/blob/main/contracts/BlurExchange.sol/#L5-L16)
[ExecutionDelegate.sol#L5-L8](https://github.com/code-423n4/2022-10-blur/blob/main/contracts/ExecutionDelegate.sol#L5-L8)
[lib/ERC1967Proxy.sol#L5-L6](https://github.com/code-423n4/2022-10-blur/blob/main/contracts/lib/ERC1967Proxy.sol#L5-L6)

Instead of this use named imports as you do on for example;
[PolicyManager.sol#L4-L7](https://github.com/code-423n4/2022-10-blur/blob/main/contracts/PolicyManager.sol#L4-L7)
[lib/EIP712.sol#L4](https://github.com/code-423n4/2022-10-blur/blob/main/contracts/lib/EIP712.sol#L4)
[BlurExchange.sol#L17-L24](https://github.com/code-423n4/2022-10-blur/blob/main/contracts/BlurExchange.sol#L17-L24)

## Add gap to reserve space on upgradeble contracts to add new variables

On import [lib/ReentrancyGuarded.sol](https://github.com/code-423n4/2022-10-blur/blob/main/contracts/lib/ReentrancyGuarded.sol) and [lib/EIP712.sol](https://github.com/code-423n4/2022-10-blur/blob/main/contracts/lib/EIP712.sol) add this lines;
```solidity
    /**
     * @dev This empty reserved space is put in place to allow future versions to add new
     * variables without shifting down storage in the inheritance chain.
     * See https://docs.openzeppelin.com/contracts/4.x/upgradeable#storage_gaps
     */
    uint256[50] private __gap;
```

In case you need to add variables for an upgrade you will have reserved space.

## On `BlurExchange.sol`

### Remove unused import `ERC20.sol`
On line [contracts/BlurExchange.sol#L8](https://github.com/code-423n4/2022-10-blur/blob/main/contracts/BlurExchange.sol#L8)

### Avoid reinvent a reentrancy guard
Openzeppellin already provides you a reentrancy guard, use this instead of [BlurExchange.sol/#L10](https://github.com/code-423n4/2022-10-blur/blob/main/contracts/BlurExchange.sol/#L10)
Replace
```
import "./lib/ReentrancyGuarded.sol";
```
with;
```
import "@openzeppelin/contracts-upgradeable/security/ReentrancyGuardUpgradeable.sol";
```
Take notes that you should also add `__ReentrancyGuard_init()` to you `initialize` function.
Using a battle test library is better, and also will be less line to audit.

Please see;
https://docs.openzeppelin.com/contracts/4.x/upgradeable
https://docs.openzeppelin.com/contracts/4.x/api/security#ReentrancyGuard

### `ERC1967Proxy` implementation its a copy of openzepellin implementation
Uso openzepellin implementation instead of copy the file.
https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/proxy/ERC1967/ERC1967Proxy.sol

```import {ERC1967Proxy} from "@openzeppelin/contracts/proxy/ERC1967/ERC1967Proxy.sol"```


### Avoid reinvent a `Pausable` feature
This lines creates are to make the contract `Pausable`;
[contracts/BlurExchange.sol#L33-L50](https://github.com/code-423n4/2022-10-blur/blob/main/contracts/BlurExchange.sol#L33-L50)

This is already implement in openzepellin on [contracts/security/PausableUpgradeable.sol](https://github.com/OpenZeppelin/openzeppelin-contracts-upgradeable/blob/master/contracts/security/PausableUpgradeable.sol) use this instead of your on implementation.

Please see;
https://docs.openzeppelin.com/contracts/4.x/upgradeable
https://docs.openzeppelin.com/contracts/4.x/api/security#Pausable

### Scientific notation may be used
For readability, it is better to use scientific notation.
On [BlurExchange.sol/#L59](https://github.com/code-423n4/2022-10-blur/blob/main/contracts/BlurExchange.sol/#L59) Replace 10000 with 1e4.

### Be explicit declaring types
On [BlurExchange.sol/#L96](https://github.com/code-423n4/2022-10-blur/blob/main/contracts/BlurExchange.sol/#L96) and [BlurExchange.sol/#L101](https://github.com/code-423n4/2022-10-blur/blob/main/contracts/BlurExchange.sol/#L101) instead of `uint` use `uint256`

### Avoid passing as reference the `chainid`
On [BlurExchange.sol/#L96](https://github.com/code-423n4/2022-10-blur/blob/main/contracts/BlurExchange.sol/#L96).
Instead of passing `chainid` as reference you could get current `chainid` using `block.chainid`
Please see;
https://docs.soliditylang.org/en/v0.8.0/units-and-global-variables.html#block-and-transaction-properties


### Missing `address(0)` checks for `_executionDelegate`, `_policyManager` and `_oracle`
By mistake any of these could be `address(0)` the could be chaged later by and admin, however is a good practice to check for `address(0)`
[BlurExchange.sol/#L98-L100](https://github.com/code-423n4/2022-10-blur/blob/main/contracts/BlurExchange.sol/#L98-L100)
Recommendation, add address(0) check;
```
require(address(_VARIABLE) != address(0), "Address cannot be zero");
```


### Missing `address(0)` for `_weth`
`_weth` variable on [contracts/BlurExchange.sol/#L97](https://github.com/code-423n4/2022-10-blur/blob/main/contracts/BlurExchange.sol/#L97) could be set as `address(0)` as mistake and there is no way to change it.
Recommendation, add address(0) check;
```
require(address(_weth) != address(0), "Address cannot be zero");
```

### If `blockRange` is `0`, `_validateSignatures` will always fail updating oracle
Inspecting the functions that set `blockRange` on [BlurExchange.sol#L117](https://github.com/code-423n4/2022-10-blur/blob/main/contracts/BlurExchange.sol#L117) and [BlurExchange.sol#L246](https://github.com/code-423n4/2022-10-blur/blob/main/contracts/BlurExchange.sol#L246) it seems that `blockRange` can be `0`.

But if you set `blockRange` to `0` the condition that checks oracle authoriozation in [line 318](https://github.com/code-423n4/2022-10-blur/blob/main/contracts/BlurExchange.sol#L318) will always fail;
`require(block.number - order.blockNumber < blockRange, "Signed block number out of range");`

Recommendation add `<=` to the require, or create a minimal blockRange required. 

### Revert if `ecrecover` is address(0)
On [BlurExchange.sol#L408](https://github.com/code-423n4/2022-10-blur/blob/main/contracts/BlurExchange.sol#L408) add a revert that triggers if the response is address(0), this means that signature its not valid.

Example, by definition `oracle` could be initialized with address(0), then you will always can pass this line (oracle validation);
[BlurExchange.sol#L392](https://github.com/code-423n4/2022-10-blur/blob/main/contracts/BlurExchange.sol#L392)
`return _recover(oracleHash, v, r, s) == oracle;`

And also you could end up stealing, because is used on `_validateSignatures` [BlurExchange.sol#L320](https://github.com/code-423n4/2022-10-blur/blob/main/contracts/BlurExchange.sol#L320) and this is also used on the main `execute` function that transfers nfts and tokens [BlurExchange.sol#L128](https://github.com/code-423n4/2022-10-blur/blob/main/contracts/BlurExchange.sol#L128)

1) avoid oracle to be address(0)
2) revert if ecrecover is address(0)
3) use openzepellin implementatio to reduce audit lines and headaches


### Avoid using low call function `ecrecover`
Use OZ library ECDSA that its battle tested to avoid classic errors.
[contracts/utils/cryptography/ECDSA.sol](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/v4.7.1/contracts/utils/cryptography/ECDSA.sol)
https://docs.openzeppelin.com/contracts/4.x/api/utils#ECDSA


### Empty revert message
On [BlurExchange.sol/#L134](https://github.com/code-423n4/2022-10-blur/blob/main/contracts/BlurExchange.sol/#L134), [BlurExchange.sol#L183](https://github.com/code-423n4/2022-10-blur/blob/main/contracts/BlurExchange.sol#L183) and [BlurExchange.sol#L452](https://github.com/code-423n4/2022-10-blur/blob/main/contracts/BlurExchange.sol#L452) there is no revert message, is very important to add a message, so the user has enough information to know the reason of failure.

### Possible DOS out of gas on `_transferFees` function loop
This loop could drain all user gas and revert;
https://github.com/code-423n4/2022-10-blur/blob/main/contracts/BlurExchange.sol#L476-L479

### No validation on `fees`
Fees can have any desiree amount. Recommendation create a threashold to avoid excessive fees.
[BlurExchange.sol/#L477](https://github.com/code-423n4/2022-10-blur/blob/main/contracts/BlurExchange.sol/#L477)

### Instead `_exist(address)` method use OZ `isContract(address)`
Openzeppellin already has checks for contract, intead of reimplementing use openzeppellin implementation;
https://docs.openzeppelin.com/contracts/4.x/api/utils#Address-isContract-address-

### Return value its not checked
The function `transferERC20` on [`BlurExchange.sol#L511`](https://github.com/code-423n4/2022-10-blur/blob/main/contracts/BlurExchange.sol#L511) returns a boolean, but its not checked in this line, add a require or remove the return.

### Use Checks-effects-interactions pattern on `execute`
Just move the `cancelledOrFilled` setting to stick to the Checks-effects-interactions pattern.
```diff
--- a/contracts/BlurExchange.sol
+++ b/contracts/BlurExchange.sol
@@ -144,6 +144,10 @@ contract BlurExchange is IBlurExchange, ReentrancyGuarded, EIP712, OwnableUpgrad

         (uint256 price, uint256 tokenId, uint256 amount, AssetType assetType) = _canMatchOrders(sell.order, buy.order);

+        /* Mark orders as filled. */
+        cancelledOrFilled[sellHash] = true;
+        cancelledOrFilled[buyHash] = true;
+
         _executeFundsTransfer(
             sell.order.trader,
             buy.order.trader,
@@ -160,10 +164,6 @@ contract BlurExchange is IBlurExchange, ReentrancyGuarded, EIP712, OwnableUpgrad
             assetType
         );

-        /* Mark orders as filled. */
-        cancelledOrFilled[sellHash] = true;
-        cancelledOrFilled[buyHash] = true;
-
         emit OrdersMatched(
             sell.order.listingTime <= buy.order.listingTime ? sell.order.trader : buy.order.trader,
             sell.order.listingTime > buy.order.listingTime ? sell.order.trader : buy.order.trader,
```

### Use OZ MerkleTree implementation instead of creating a new one
Instead of your own merkle tree lib, [BlurExchange.sol#L12](https://github.com/code-423n4/2022-10-blur/blob/main/contracts/BlurExchange.sol#L12)
Use openzeppelin implementation;
https://docs.openzeppelin.com/contracts/4.x/api/utils#MerkleProof

### Use OZ eip712 instead of creating your onw implementation
Use openzepellin implementation, and save a lot of possible bugs by writing your onw implementation;
Code is in;
https://github.com/OpenZeppelin/openzeppelin-contracts-upgradeable/blob/master/contracts/utils/cryptography/EIP712Upgradeable.sol

Just;
```
import "@openzeppelin/contracts-upgradeable/utils/cryptography/EIP712Upgradeable.sol";
```
https://docs.openzeppelin.com/contracts/4.x/api/utils#EIP712
(docs could be old, its not on draft anymore)

## Contract EIP712 should be marked as abstract
contract [EIP712](https://github.com/code-423n4/2022-10-blur/blob/main/contracts/lib/EIP712.sol#L10) should be abstract

