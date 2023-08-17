## Tags

- bug
- G (Gas Optimization)
- selected for report

# [Gas Optimizations](https://github.com/code-423n4/2022-10-blur-findings/issues/550) 

## Summary

### Gas Optimizations
| |Issue|Instances|Total Gas Saved|
|-|:-|:-:|:-:|
| [G&#x2011;01] | State variables that never change should be declared `immutable` or `constant` | 1 | 2097 |
| [G&#x2011;02] | Multiple `address`/ID mappings can be combined into a single `mapping` of an `address`/ID to a `struct`, where appropriate | 1 | 8400 |
| [G&#x2011;03] | Structs can be packed into fewer storage slots | 1 | - |
| [G&#x2011;04] | Using `calldata` instead of `memory` for read-only arguments in `external` functions saves gas | 1 | 120 |
| [G&#x2011;05] | State variables should be cached in stack variables rather than re-reading them from storage | 1 | 97 |
| [G&#x2011;06] | The result of function calls should be cached rather than re-calling the function | 1 | 17 |
| [G&#x2011;07] | `internal` functions only called once can be inlined to save gas | 10 | 200 |
| [G&#x2011;08] | Add `unchecked {}` for subtractions where the operands cannot underflow because of a previous `require()` or `if`-statement | 1 | 85 |
| [G&#x2011;09] | `<array>.length` should not be looked up in every loop of a `for`-loop | 4 | 12 |
| [G&#x2011;10] | `++i`/`i++` should be `unchecked{++i}`/`unchecked{i++}` when it is not possible for them to overflow, as is the case when used in `for`- and `while`-loops | 5 | 300 |
| [G&#x2011;11] | `require()`/`revert()` strings longer than 32 bytes cost extra gas | 2 | - |
| [G&#x2011;12] | Optimize names to save gas | 3 | 66 |
| [G&#x2011;13] | Using `bool`s for storage incurs overhead | 4 | 51300 |
| [G&#x2011;14] | `++i` costs less gas than `i++`, especially when it's used in `for`-loops (`--i`/`i--` too) | 5 | 25 |
| [G&#x2011;15] | Using `private` rather than `public` for constants, saves gas | 7 | - |
| [G&#x2011;16] | Don't compare boolean expressions to boolean literals | 5 | 45 |
| [G&#x2011;17] | Use custom errors rather than `revert()`/`require()` strings to save gas | 23 | - |
| [G&#x2011;18] | Functions guaranteed to revert when called by normal users can be marked `payable` | 11 | 231 |

Total: 86 instances over 18 issues with **62,995 gas** saved

Gas totals use lower bounds of ranges and count two iterations of each `for`-loop. All values above are runtime, not deployment, values; deployment values are listed in the individual issue descriptions



## Gas Optimizations

### [G&#x2011;01]  State variables that never change should be declared `immutable` or `constant`
Avoids a Gsset (**20000 gas**) in the constructor, and replaces the first access in each transaction (Gcoldsload - **2100 gas**) and each access thereafter (Gwarmacces - **100 gas**) with a `PUSH32` (**3 gas**). 

While it's not possible to use `immutable` because the contract is UUPS, the variable can be declared as a hard-coded `constant` and get the same gas savings

*There is 1 instance of this issue:*
```solidity
File: /contracts/BlurExchange.sol

509          } else if (paymentToken == weth) {
510              /* Transfer funds in WETH. */
511:             executionDelegate.transferERC20(weth, from, to, amount);

```
https://github.com/code-423n4/2022-10-blur/blob/2fdaa6e13b544c8c11d1c022a575f16c3a72e3bf/contracts/BlurExchange.sol#L509-L511

### [G&#x2011;02]  Multiple `address`/ID mappings can be combined into a single `mapping` of an `address`/ID to a `struct`, where appropriate
Saves a storage slot for the mapping. Depending on the circumstances and sizes of types, can avoid a Gsset (**20000 gas**) per mapping combined. Reads and subsequent writes can also be cheaper when a function requires both values and they both fit in the same storage slot. Finally, if both fields are accessed in the same function, can save **~42 gas per access** due to [not having to recalculate the key's keccak256 hash](https://gist.github.com/IllIllI000/ec23a57daa30a8f8ca8b9681c8ccefb0) (Gkeccak256 - 30 gas) and that calculation's associated stack operations.

All functions that check `contracts` also check `revokedApproval`, which means if the modifier is changed to check both, both the ~42 gas and the ~2100 gas savings get applied. There are four instances of the `approvedContract()` modifier, so making such a change saves approximately **8,400 gas**
*There is 1 instance of this issue:*
```solidity
File: contracts/ExecutionDelegate.sol

18        mapping(address => bool) public contracts;
19:       mapping(address => bool) public revokedApproval;

```
https://github.com/code-423n4/2022-10-blur/blob/2fdaa6e13b544c8c11d1c022a575f16c3a72e3bf/contracts/ExecutionDelegate.sol#L18-L19

### [G&#x2011;03]  Structs can be packed into fewer storage slots
Each slot saved can avoid an extra Gsset (**20000 gas**) for the first setting of the struct. Subsequent reads as well as writes have smaller gas savings

*There is 1 instance of this issue:*
```solidity
File: contracts/lib/OrderStructs.sol

/// @audit Variable ordering with 6 slots instead of the current 7:
///           user-defined(32):order, bytes32(32):r, bytes32(32):s, bytes(32):extraSignature, uint256(32):blockNumber, uint8(1):v, uint8(1):signatureVersion
30    struct Input {
31        Order order;
32        uint8 v;
33        bytes32 r;
34        bytes32 s;
35        bytes extraSignature;
36        SignatureVersion signatureVersion;
37        uint256 blockNumber;
38:   }

```
https://github.com/code-423n4/2022-10-blur/blob/2fdaa6e13b544c8c11d1c022a575f16c3a72e3bf/contracts/lib/OrderStructs.sol#L30-L38

### [G&#x2011;04]  Using `calldata` instead of `memory` for read-only arguments in `external` functions saves gas
When a function with a `memory` array is called externally, the `abi.decode()` step has to use a for-loop to copy each index of the `calldata` to the `memory` index. **Each iteration of this for-loop costs at least 60 gas** (i.e. `60 * <mem_array>.length`). Using `calldata` directly, obliviates the need for such a loop in the contract code and runtime execution. Note that even if an interface defines a function as having `memory` arguments, it's still valid for implementation contracs to use `calldata` arguments instead. 

If the array is passed to an `internal` function which passes the array to another internal function where the array is modified and therefore `memory` is used in the `external` call, it's still more gass-efficient to use `calldata` when the `external` function uses modifiers, since the modifiers may prevent the internal functions from being called. Structs have the same overhead as an array of length one

Note that I've also flagged instances where the function is `public` but can be marked as `external` since it's not called by the contract, and cases where a constructor is involved

*There is 1 instance of this issue:*
```solidity
File: contracts/lib/MerkleVerifier.sol

/// @audit proof
17        function _verifyProof(
18            bytes32 leaf,
19            bytes32 root,
20:           bytes32[] memory proof

```
https://github.com/code-423n4/2022-10-blur/blob/2fdaa6e13b544c8c11d1c022a575f16c3a72e3bf/contracts/lib/MerkleVerifier.sol#L17-L20

### [G&#x2011;05]  State variables should be cached in stack variables rather than re-reading them from storage
The instances below point to the second+ access of a state variable within a function. Caching of a state variable replaces each Gwarmaccess (**100 gas**) with a much cheaper stack read. Other less obvious fixes/optimizations include having local memory caches of state variable structs, or having local caches of state variable contracts/addresses.

*There is 1 instance of this issue:*
```solidity
File: contracts/BlurExchange.sol

/// @audit weth on line 509
511:              executionDelegate.transferERC20(weth, from, to, amount);

```
https://github.com/code-423n4/2022-10-blur/blob/2fdaa6e13b544c8c11d1c022a575f16c3a72e3bf/contracts/BlurExchange.sol#L511

### [G&#x2011;06]  The result of function calls should be cached rather than re-calling the function
The instances below point to the second+ call of the function within a single function

*There is 1 instance of this issue:*
```solidity
File: contracts/PolicyManager.sol

/// @audit _whitelistedPolicies.length() on line 71
72:               length = _whitelistedPolicies.length() - cursor;

```
https://github.com/code-423n4/2022-10-blur/blob/2fdaa6e13b544c8c11d1c022a575f16c3a72e3bf/contracts/PolicyManager.sol#L72

### [G&#x2011;07]  `internal` functions only called once can be inlined to save gas
Not inlining costs **20 to 40 gas** because of two extra `JUMP` instructions and additional stack operations needed for function calls.

*There are 10 instances of this issue:*
```solidity
File: contracts/BlurExchange.sol

278       function _canSettleOrder(uint256 listingTime, uint256 expirationTime)
279           view
280           internal
281:          returns (bool)

344       function _validateUserAuthorization(
345           bytes32 orderHash,
346           address trader,
347           uint8 v,
348           bytes32 r,
349           bytes32 s,
350           SignatureVersion signatureVersion,
351           bytes calldata extraSignature
352:      ) internal view returns (bool) {

375       function _validateOracleAuthorization(
376           bytes32 orderHash,
377           SignatureVersion signatureVersion,
378           bytes calldata extraSignature,
379           uint256 blockNumber
380:      ) internal view returns (bool) {

416       function _canMatchOrders(Order calldata sell, Order calldata buy)
417           internal
418           view
419:          returns (uint256 price, uint256 tokenId, uint256 amount, AssetType assetType)

444       function _executeFundsTransfer(
445           address seller,
446           address buyer,
447           address paymentToken,
448           Fee[] calldata fees,
449:          uint256 price

469       function _transferFees(
470           Fee[] calldata fees,
471           address paymentToken,
472           address from,
473           uint256 price
474:      ) internal returns (uint256) {

525       function _executeTokenTransfer(
526           address collection,
527           address from,
528           address to,
529           uint256 tokenId,
530           uint256 amount,
531:          AssetType assetType

548       function _exists(address what)
549           internal
550           view
551:          returns (bool)

```
https://github.com/code-423n4/2022-10-blur/blob/2fdaa6e13b544c8c11d1c022a575f16c3a72e3bf/contracts/BlurExchange.sol#L278-L281

```solidity
File: contracts/lib/EIP712.sol

55        function _hashFee(Fee calldata fee)
56            internal 
57            pure
58:           returns (bytes32)

69        function _packFees(Fee[] calldata fees)
70            internal
71            pure
72:           returns (bytes32)

```
https://github.com/code-423n4/2022-10-blur/blob/2fdaa6e13b544c8c11d1c022a575f16c3a72e3bf/contracts/lib/EIP712.sol#L55-L58

### [G&#x2011;08]  Add `unchecked {}` for subtractions where the operands cannot underflow because of a previous `require()` or `if`-statement
`require(a <= b); x = b - a` => `require(a <= b); unchecked { x = b - a }`

*There is 1 instance of this issue:*
```solidity
File: contracts/BlurExchange.sol

/// @audit require() on line 482
485:          uint256 receiveAmount = price - totalFee;

```
https://github.com/code-423n4/2022-10-blur/blob/2fdaa6e13b544c8c11d1c022a575f16c3a72e3bf/contracts/BlurExchange.sol#L485

### [G&#x2011;09]  `<array>.length` should not be looked up in every loop of a `for`-loop
The overheads outlined below are _PER LOOP_, excluding the first loop
* storage arrays incur a Gwarmaccess (**100 gas**)
* memory arrays use `MLOAD` (**3 gas**)
* calldata arrays use `CALLDATALOAD` (**3 gas**)

Caching the length changes each of these to a `DUP<N>` (**3 gas**), and gets rid of the extra `DUP<N>` needed to store the stack offset

*There are 4 instances of this issue:*
```solidity
File: contracts/BlurExchange.sol

199:          for (uint8 i = 0; i < orders.length; i++) {

476:          for (uint8 i = 0; i < fees.length; i++) {

```
https://github.com/code-423n4/2022-10-blur/blob/2fdaa6e13b544c8c11d1c022a575f16c3a72e3bf/contracts/BlurExchange.sol#L199

```solidity
File: contracts/lib/EIP712.sol

77:           for (uint256 i = 0; i < fees.length; i++) {

```
https://github.com/code-423n4/2022-10-blur/blob/2fdaa6e13b544c8c11d1c022a575f16c3a72e3bf/contracts/lib/EIP712.sol#L77

```solidity
File: contracts/lib/MerkleVerifier.sol

38:           for (uint256 i = 0; i < proof.length; i++) {

```
https://github.com/code-423n4/2022-10-blur/blob/2fdaa6e13b544c8c11d1c022a575f16c3a72e3bf/contracts/lib/MerkleVerifier.sol#L38

### [G&#x2011;10]  `++i`/`i++` should be `unchecked{++i}`/`unchecked{i++}` when it is not possible for them to overflow, as is the case when used in `for`- and `while`-loops
The `unchecked` keyword is new in solidity version 0.8.0, so this only applies to that version or higher, which these instances are. This saves **30-40 gas [per loop](https://gist.github.com/hrkrshnn/ee8fabd532058307229d65dcd5836ddc#the-increment-in-for-loop-post-condition-can-be-made-unchecked)**

*There are 5 instances of this issue:*
```solidity
File: contracts/BlurExchange.sol

199:          for (uint8 i = 0; i < orders.length; i++) {

476:          for (uint8 i = 0; i < fees.length; i++) {

```
https://github.com/code-423n4/2022-10-blur/blob/2fdaa6e13b544c8c11d1c022a575f16c3a72e3bf/contracts/BlurExchange.sol#L199

```solidity
File: contracts/lib/EIP712.sol

77:           for (uint256 i = 0; i < fees.length; i++) {

```
https://github.com/code-423n4/2022-10-blur/blob/2fdaa6e13b544c8c11d1c022a575f16c3a72e3bf/contracts/lib/EIP712.sol#L77

```solidity
File: contracts/lib/MerkleVerifier.sol

38:           for (uint256 i = 0; i < proof.length; i++) {

```
https://github.com/code-423n4/2022-10-blur/blob/2fdaa6e13b544c8c11d1c022a575f16c3a72e3bf/contracts/lib/MerkleVerifier.sol#L38

```solidity
File: contracts/PolicyManager.sol

77:           for (uint256 i = 0; i < length; i++) {

```
https://github.com/code-423n4/2022-10-blur/blob/2fdaa6e13b544c8c11d1c022a575f16c3a72e3bf/contracts/PolicyManager.sol#L77

### [G&#x2011;11]  `require()`/`revert()` strings longer than 32 bytes cost extra gas
Each extra memory word of bytes past the original 32 [incurs an MSTORE](https://gist.github.com/hrkrshnn/ee8fabd532058307229d65dcd5836ddc#consider-having-short-revert-strings) which costs **3 gas**

*There are 2 instances of this issue:*
```solidity
File: contracts/BlurExchange.sol

482:          require(totalFee <= price, "Total amount of fees are more than the price");

```
https://github.com/code-423n4/2022-10-blur/blob/2fdaa6e13b544c8c11d1c022a575f16c3a72e3bf/contracts/BlurExchange.sol#L482

```solidity
File: contracts/ExecutionDelegate.sol

22:           require(contracts[msg.sender], "Contract is not approved to make transfers");

```
https://github.com/code-423n4/2022-10-blur/blob/2fdaa6e13b544c8c11d1c022a575f16c3a72e3bf/contracts/ExecutionDelegate.sol#L22

### [G&#x2011;12]  Optimize names to save gas
`public`/`external` function names and `public` member variable names can be optimized to save gas. See [this](https://gist.github.com/IllIllI000/a5d8b486a8259f9f77891a919febd1a9) link for an example of how it works. Below are the interfaces/abstract contracts that can be optimized so that the most frequently-called functions use the least amount of gas possible during method lookup. Method IDs that have two leading zero bytes can save **128 gas** each during deployment, and renaming functions to have lower method IDs will save **22 gas** per call, [per sorted position shifted](https://medium.com/joyso/solidity-how-does-function-name-affect-gas-consumption-in-smart-contract-47d270d8ac92)

*There are 3 instances of this issue:*
```solidity
File: contracts/BlurExchange.sol

/// @audit open(), close(), initialize(), execute(), cancelOrder(), cancelOrders(), incrementNonce(), setExecutionDelegate(), setPolicyManager(), setOracle(), setBlockRange()
30:   contract BlurExchange is IBlurExchange, ReentrancyGuarded, EIP712, OwnableUpgradeable, UUPSUpgradeable {

```
https://github.com/code-423n4/2022-10-blur/blob/2fdaa6e13b544c8c11d1c022a575f16c3a72e3bf/contracts/BlurExchange.sol#L30

```solidity
File: contracts/ExecutionDelegate.sol

/// @audit approveContract(), denyContract(), revokeApproval(), grantApproval(), transferERC721Unsafe(), transferERC721(), transferERC1155(), transferERC20()
16:   contract ExecutionDelegate is IExecutionDelegate, Ownable {

```
https://github.com/code-423n4/2022-10-blur/blob/2fdaa6e13b544c8c11d1c022a575f16c3a72e3bf/contracts/ExecutionDelegate.sol#L16

```solidity
File: contracts/lib/MerkleVerifier.sol

/// @audit _verifyProof(), _computeRoot()
8:    library MerkleVerifier {

```
https://github.com/code-423n4/2022-10-blur/blob/2fdaa6e13b544c8c11d1c022a575f16c3a72e3bf/contracts/lib/MerkleVerifier.sol#L8

### [G&#x2011;13]  Using `bool`s for storage incurs overhead
```solidity
    // Booleans are more expensive than uint256 or any type that takes up a full
    // word because each write operation emits an extra SLOAD to first read the
    // slot's contents, replace the bits taken up by the boolean, and then write
    // back. This is the compiler's defense against contract upgrades and
    // pointer aliasing, and it cannot be disabled.
```
https://github.com/OpenZeppelin/openzeppelin-contracts/blob/58f635312aa21f947cae5f8578638a85aa2519f5/contracts/security/ReentrancyGuard.sol#L23-L27
Use `uint256(1)` and `uint256(2)` for true/false to avoid a Gwarmaccess (**[100 gas](https://gist.github.com/IllIllI000/1b70014db712f8572a72378321250058)**) for the extra SLOAD, and to avoid Gsset (**20000 gas**) when changing from `false` to `true`, after having been `true` in the past

*There are 4 instances of this issue:*
```solidity
File: contracts/BlurExchange.sol

/// @audit excluding this one from the gas count since it's not changed after being set
71:       mapping(bytes32 => bool) public cancelledOrFilled;

```
https://github.com/code-423n4/2022-10-blur/blob/2fdaa6e13b544c8c11d1c022a575f16c3a72e3bf/contracts/BlurExchange.sol#L71

```solidity
File: contracts/ExecutionDelegate.sol

18:       mapping(address => bool) public contracts;

19:       mapping(address => bool) public revokedApproval;

```
https://github.com/code-423n4/2022-10-blur/blob/2fdaa6e13b544c8c11d1c022a575f16c3a72e3bf/contracts/ExecutionDelegate.sol#L18

```solidity
File: contracts/lib/ReentrancyGuarded.sol

10:       bool reentrancyLock = false;

```
https://github.com/code-423n4/2022-10-blur/blob/2fdaa6e13b544c8c11d1c022a575f16c3a72e3bf/contracts/lib/ReentrancyGuarded.sol#L10

### [G&#x2011;14]  `++i` costs less gas than `i++`, especially when it's used in `for`-loops (`--i`/`i--` too)
Saves **5 gas per loop**

*There are 5 instances of this issue:*
```solidity
File: contracts/BlurExchange.sol

199:          for (uint8 i = 0; i < orders.length; i++) {

476:          for (uint8 i = 0; i < fees.length; i++) {

```
https://github.com/code-423n4/2022-10-blur/blob/2fdaa6e13b544c8c11d1c022a575f16c3a72e3bf/contracts/BlurExchange.sol#L199

```solidity
File: contracts/lib/EIP712.sol

77:           for (uint256 i = 0; i < fees.length; i++) {

```
https://github.com/code-423n4/2022-10-blur/blob/2fdaa6e13b544c8c11d1c022a575f16c3a72e3bf/contracts/lib/EIP712.sol#L77

```solidity
File: contracts/lib/MerkleVerifier.sol

38:           for (uint256 i = 0; i < proof.length; i++) {

```
https://github.com/code-423n4/2022-10-blur/blob/2fdaa6e13b544c8c11d1c022a575f16c3a72e3bf/contracts/lib/MerkleVerifier.sol#L38

```solidity
File: contracts/PolicyManager.sol

77:           for (uint256 i = 0; i < length; i++) {

```
https://github.com/code-423n4/2022-10-blur/blob/2fdaa6e13b544c8c11d1c022a575f16c3a72e3bf/contracts/PolicyManager.sol#L77

### [G&#x2011;15]  Using `private` rather than `public` for constants, saves gas
If needed, the values can be read from the verified contract source code, or if there are multiple values there can be a single getter function that [returns a tuple](https://github.com/code-423n4/2022-08-frax/blob/90f55a9ce4e25bceed3a74290b854341d8de6afa/src/contracts/FraxlendPair.sol#L156-L178) of the values of all currently-public constants. Saves **3406-3606 gas** in deployment gas due to the compiler not having to create non-payable getter functions for deployment calldata, not having to store the bytes of the value outside of where it's used, and not adding another entry to the method ID table

*There are 7 instances of this issue:*
```solidity
File: contracts/BlurExchange.sol

57:       string public constant name = "Blur Exchange";

58:       string public constant version = "1.0";

59:       uint256 public constant INVERSE_BASIS_POINT = 10000;

```
https://github.com/code-423n4/2022-10-blur/blob/2fdaa6e13b544c8c11d1c022a575f16c3a72e3bf/contracts/BlurExchange.sol#L57

```solidity
File: contracts/lib/EIP712.sol

20        bytes32 constant public FEE_TYPEHASH = keccak256(
21            "Fee(uint16 rate,address recipient)"
22:       );

23        bytes32 constant public ORDER_TYPEHASH = keccak256(
24            "Order(address trader,uint8 side,address matchingPolicy,address collection,uint256 tokenId,uint256 amount,address paymentToken,uint256 price,uint256 listingTime,uint256 expirationTime,Fee[] fees,uint256 salt,bytes extraParams,uint256 nonce)Fee(uint16 rate,address recipient)"
25:       );

26        bytes32 constant public ORACLE_ORDER_TYPEHASH = keccak256(
27            "OracleOrder(Order order,uint256 blockNumber)Fee(uint16 rate,address recipient)Order(address trader,uint8 side,address matchingPolicy,address collection,uint256 tokenId,uint256 amount,address paymentToken,uint256 price,uint256 listingTime,uint256 expirationTime,Fee[] fees,uint256 salt,bytes extraParams,uint256 nonce)"
28:       );

29        bytes32 constant public ROOT_TYPEHASH = keccak256(
30            "Root(bytes32 root)"
31:       );

```
https://github.com/code-423n4/2022-10-blur/blob/2fdaa6e13b544c8c11d1c022a575f16c3a72e3bf/contracts/lib/EIP712.sol#L20-L22

### [G&#x2011;16]  Don't compare boolean expressions to boolean literals
`if (<x> == true)` => `if (<x>)`, `if (<x> == false)` => `if (!<x>)`

*There are 5 instances of this issue:*
```solidity
File: contracts/BlurExchange.sol

267:              (cancelledOrFilled[orderHash] == false) &&

```
https://github.com/code-423n4/2022-10-blur/blob/2fdaa6e13b544c8c11d1c022a575f16c3a72e3bf/contracts/BlurExchange.sol#L267

```solidity
File: contracts/ExecutionDelegate.sol

77:           require(revokedApproval[from] == false, "User has revoked approval");

92:           require(revokedApproval[from] == false, "User has revoked approval");

108:          require(revokedApproval[from] == false, "User has revoked approval");

124:          require(revokedApproval[from] == false, "User has revoked approval");

```
https://github.com/code-423n4/2022-10-blur/blob/2fdaa6e13b544c8c11d1c022a575f16c3a72e3bf/contracts/ExecutionDelegate.sol#L77

### [G&#x2011;17]  Use custom errors rather than `revert()`/`require()` strings to save gas
Custom errors are available from solidity version 0.8.4. Custom errors save [**~50 gas**](https://gist.github.com/IllIllI000/ad1bd0d29a0101b25e57c293b4b0c746) each time they're hit by [avoiding having to allocate and store the revert string](https://blog.soliditylang.org/2021/04/21/custom-errors/#errors-in-depth). Not defining the strings also save deployment gas

*There are 23 instances of this issue:*
```solidity
File: contracts/BlurExchange.sol

36:           require(isOpen == 1, "Closed");

139:          require(_validateOrderParameters(sell.order, sellHash), "Sell has invalid parameters");

140:          require(_validateOrderParameters(buy.order, buyHash), "Buy has invalid parameters");

142:          require(_validateSignatures(sell, sellHash), "Sell failed authorization");

143:          require(_validateSignatures(buy, buyHash), "Buy failed authorization");

219:          require(address(_executionDelegate) != address(0), "Address cannot be zero");

228:          require(address(_policyManager) != address(0), "Address cannot be zero");

237:          require(_oracle != address(0), "Address cannot be zero");

318:              require(block.number - order.blockNumber < blockRange, "Signed block number out of range");

407:          require(v == 27 || v == 28, "Invalid v parameter");

424:              require(policyManager.isPolicyWhitelisted(sell.matchingPolicy), "Policy is not whitelisted");

428:              require(policyManager.isPolicyWhitelisted(buy.matchingPolicy), "Policy is not whitelisted");

431:          require(canMatch, "Orders cannot be matched");

482:          require(totalFee <= price, "Total amount of fees are more than the price");

534:          require(_exists(collection), "Collection does not exist");

```
https://github.com/code-423n4/2022-10-blur/blob/2fdaa6e13b544c8c11d1c022a575f16c3a72e3bf/contracts/BlurExchange.sol#L36

```solidity
File: contracts/ExecutionDelegate.sol

22:           require(contracts[msg.sender], "Contract is not approved to make transfers");

77:           require(revokedApproval[from] == false, "User has revoked approval");

92:           require(revokedApproval[from] == false, "User has revoked approval");

108:          require(revokedApproval[from] == false, "User has revoked approval");

124:          require(revokedApproval[from] == false, "User has revoked approval");

```
https://github.com/code-423n4/2022-10-blur/blob/2fdaa6e13b544c8c11d1c022a575f16c3a72e3bf/contracts/ExecutionDelegate.sol#L22

```solidity
File: contracts/lib/ReentrancyGuarded.sol

14:           require(!reentrancyLock, "Reentrancy detected");

```
https://github.com/code-423n4/2022-10-blur/blob/2fdaa6e13b544c8c11d1c022a575f16c3a72e3bf/contracts/lib/ReentrancyGuarded.sol#L14

```solidity
File: contracts/PolicyManager.sol

26:           require(!_whitelistedPolicies.contains(policy), "Already whitelisted");

37:           require(_whitelistedPolicies.contains(policy), "Not whitelisted");

```
https://github.com/code-423n4/2022-10-blur/blob/2fdaa6e13b544c8c11d1c022a575f16c3a72e3bf/contracts/PolicyManager.sol#L26

### [G&#x2011;18]  Functions guaranteed to revert when called by normal users can be marked `payable`
If a function modifier such as `onlyOwner` is used, the function will revert if a normal user tries to pay the function. Marking the function as `payable` will lower the gas cost for legitimate callers because the compiler will not include checks for whether a payment was provided. The extra opcodes avoided are 
`CALLVALUE`(2),`DUP1`(3),`ISZERO`(3),`PUSH2`(3),`JUMPI`(10),`PUSH1`(3),`DUP1`(3),`REVERT`(0),`JUMPDEST`(1),`POP`(2), which costs an average of about **21 gas per call** to the function, in addition to the extra deployment cost

*There are 11 instances of this issue:*
```solidity
File: contracts/BlurExchange.sol

43:       function open() external onlyOwner {

47:       function close() external onlyOwner {

53:       function _authorizeUpgrade(address) internal override onlyOwner {}

215       function setExecutionDelegate(IExecutionDelegate _executionDelegate)
216           external
217:          onlyOwner

224       function setPolicyManager(IPolicyManager _policyManager)
225           external
226:          onlyOwner

233       function setOracle(address _oracle)
234           external
235:          onlyOwner

242       function setBlockRange(uint256 _blockRange)
243           external
244:          onlyOwner

```
https://github.com/code-423n4/2022-10-blur/blob/2fdaa6e13b544c8c11d1c022a575f16c3a72e3bf/contracts/BlurExchange.sol#L43

```solidity
File: contracts/ExecutionDelegate.sol

36:       function approveContract(address _contract) onlyOwner external {

45:       function denyContract(address _contract) onlyOwner external {

```
https://github.com/code-423n4/2022-10-blur/blob/2fdaa6e13b544c8c11d1c022a575f16c3a72e3bf/contracts/ExecutionDelegate.sol#L36

```solidity
File: contracts/PolicyManager.sol

25:       function addPolicy(address policy) external override onlyOwner {

36:       function removePolicy(address policy) external override onlyOwner {

```
https://github.com/code-423n4/2022-10-blur/blob/2fdaa6e13b544c8c11d1c022a575f16c3a72e3bf/contracts/PolicyManager.sol#L25

