## Tags

- bug
- G (Gas Optimization)
- old-submission-method
- selected for report

# [Gas Optimizations](https://github.com/code-423n4/2022-09-artgobblers-findings/issues/324) 

## Summary

### Gas Optimizations
| |Issue|Instances|Total Gas Saved|
|-|:-|:-:|:-:|
| [G&#x2011;01] | Save gas by not updating seed | 1 | 2897 |
| [G&#x2011;02] | State variables only set in the constructor should be declared `immutable` | 7 | 12582 |
| [G&#x2011;03] | Avoid contract existence checks by using solidity version 0.8.10 or later | 11 | 1100 |
| [G&#x2011;04] | State variables should be cached in stack variables rather than re-reading them from storage | 4 | 388 |
| [G&#x2011;05] | Multiple accesses of a mapping/array should use a local variable cache | 25 | 1050 |
| [G&#x2011;06] | `<x> += <y>` costs more gas than `<x> = <x> + <y>` for state variables | 2 | 226 |
| [G&#x2011;07] | `<array>.length` should not be looked up in every loop of a `for`-loop | 2 | 6 |
| [G&#x2011;08] | Optimize names to save gas | 10 | 220 |
| [G&#x2011;09] | Using `bool`s for storage incurs overhead | 5 | 68400 |
| [G&#x2011;10] | Use a more recent version of solidity | 22 | - |
| [G&#x2011;11] | `++i` costs less gas than `i++`, especially when it's used in `for`-loops (`--i`/`i--` too) | 9 | 45 |
| [G&#x2011;12] | Using `private` rather than `public` for constants, saves gas | 18 | - |
| [G&#x2011;13] | Division by two should use bit shifting | 1 | 20 |
| [G&#x2011;14] | Stack variable used as a cheaper cache for a state variable is only used once | 2 | 6 |
| [G&#x2011;15] | Use custom errors rather than `revert()`/`require()` strings to save gas | 36 | - |
| [G&#x2011;16] | Functions guaranteed to revert when called by normal users can be marked `payable` | 3 | 63 |

Total: 158 instances over 16 issues with **87003 gas** saved

Gas totals use lower bounds of ranges and count two iterations of each `for`-loop. All values above are runtime, not deployment, values; deployment values are listed in the individual issue descriptions


## Gas Optimizations

### [G&#x2011;01]  Save gas by not updating seed
The code that determines a set of random values based on the seed does not follow Chainlink's stated [best practice](https://docs.chain.link/docs/vrf/v1/best-practices/#getting-multiple-random-numbers) for doing so. By updating the seed rather than including a counter in what's hashed (e.g. `currentId`), the code incurs an unnecessary Gsreset **2900 gas**.

*There is 1 instance of this issue:*
```solidity
File: /src/ArtGobblers.sol

664                  // Update the random seed to choose a new distance for the next iteration.
665                  // It is critical that we cast to uint64 here, as otherwise the random seed
666                  // set after calling revealGobblers(1) thrice would differ from the seed set
667                  // after calling revealGobblers(3) a single time. This would enable an attacker
668                  // to choose from a number of different seeds and use whichever is most favorable.
669                  // Equivalent to randomSeed = uint64(uint256(keccak256(abi.encodePacked(randomSeed))))
670                  assembly {
671                      mstore(0, randomSeed) // Store the random seed in scratch space.
672  
673                      // Moduloing by 1 << 64 (2 ** 64) is equivalent to a uint64 cast.
674                      randomSeed := mod(keccak256(0, 32), shl(64, 1))
675:                 }

```
https://github.com/code-423n4/2022-09-artgobblers/blob/d2087c5a8a6a4f1b9784520e7fe75afa3a9cbdbe/src/ArtGobblers.sol#L664-L675

### [G&#x2011;02]  State variables only set in the constructor should be declared `immutable`
Avoids a Gsset (**20000 gas**) in the constructor, and replaces the first access in each transaction (Gcoldsload - **2100 gas**) and each access thereafter (Gwarmacces - **100 gas**) with a `PUSH32` (**3 gas**). 

While `string`s are not value types, and therefore cannot be `immutable`/`constant` if not hard-coded outside of the constructor, the same behavior can be achieved by making the current contract `abstract` with `virtual` functions for the `string` accessors, and having a child contract override the functions with the hard-coded implementation-specific values.

*There are 7 instances of this issue:*
```solidity
File: src/ArtGobblers.sol

/// @audit UNREVEALED_URI (constructor)
321:          UNREVEALED_URI = _unrevealedUri;

/// @audit UNREVEALED_URI (access)
702:          if (gobblerId <= currentNonLegendaryId) return UNREVEALED_URI;

/// @audit BASE_URI (constructor)
320:          BASE_URI = _baseUri;

/// @audit BASE_URI (access)
698:              return string.concat(BASE_URI, uint256(getGobblerData[gobblerId].idx).toString());

/// @audit BASE_URI (access)
709:              return string.concat(BASE_URI, gobblerId.toString());

```
https://github.com/code-423n4/2022-09-artgobblers/blob/d2087c5a8a6a4f1b9784520e7fe75afa3a9cbdbe/src/ArtGobblers.sol#L321

```solidity
File: src/Pages.sol

/// @audit BASE_URI (constructor)
183:          BASE_URI = _baseUri;

/// @audit BASE_URI (access)
268:          return string.concat(BASE_URI, pageId.toString());

```
https://github.com/code-423n4/2022-09-artgobblers/blob/d2087c5a8a6a4f1b9784520e7fe75afa3a9cbdbe/src/Pages.sol#L183

### [G&#x2011;03]  Avoid contract existence checks by using solidity version 0.8.10 or later
Prior to 0.8.10 the compiler inserted extra code, including `EXTCODESIZE` (**100 gas**), to check for contract existence for external calls. In more recent solidity versions, the compiler will not insert these checks if the external call has a return value

*There are 11 instances of this issue:*
```solidity
File: src/ArtGobblers.sol

/// @audit toString()
698:              return string.concat(BASE_URI, uint256(getGobblerData[gobblerId].idx).toString());

```
https://github.com/code-423n4/2022-09-artgobblers/blob/d2087c5a8a6a4f1b9784520e7fe75afa3a9cbdbe/src/ArtGobblers.sol#L698

```solidity
File: lib/solmate/src/tokens/ERC721.sol

/// @audit onERC721Received()
120:                  ERC721TokenReceiver(to).onERC721Received(msg.sender, from, id, "") ==

/// @audit onERC721Received()
136:                  ERC721TokenReceiver(to).onERC721Received(msg.sender, from, id, data) ==

/// @audit onERC721Received()
198:                  ERC721TokenReceiver(to).onERC721Received(msg.sender, address(0), id, "") ==

/// @audit onERC721Received()
213:                  ERC721TokenReceiver(to).onERC721Received(msg.sender, address(0), id, data) ==

```
https://github.com/transmissions11/solmate/blob/26572802743101f160f2d07556edfc162896115e/src/tokens/ERC721.sol#L120

```solidity
File: src/utils/token/GobblersERC1155B.sol

/// @audit onERC1155Received()
150:                  ERC1155TokenReceiver(to).onERC1155Received(msg.sender, address(0), id, 1, data) ==

/// @audit onERC1155BatchReceived()
186:                  ERC1155TokenReceiver(to).onERC1155BatchReceived(msg.sender, address(0), ids, amounts, data) ==

```
https://github.com/code-423n4/2022-09-artgobblers/blob/d2087c5a8a6a4f1b9784520e7fe75afa3a9cbdbe/src/utils/token/GobblersERC1155B.sol#L150

```solidity
File: src/utils/token/GobblersERC721.sol

/// @audit onERC721Received()
123:                  ERC721TokenReceiver(to).onERC721Received(msg.sender, from, id, "") ==

/// @audit onERC721Received()
139:                  ERC721TokenReceiver(to).onERC721Received(msg.sender, from, id, data) ==

```
https://github.com/code-423n4/2022-09-artgobblers/blob/d2087c5a8a6a4f1b9784520e7fe75afa3a9cbdbe/src/utils/token/GobblersERC721.sol#L123

```solidity
File: src/utils/token/PagesERC721.sol

/// @audit onERC721Received()
136:                  ERC721TokenReceiver(to).onERC721Received(msg.sender, from, id, "") ==

/// @audit onERC721Received()
152:                  ERC721TokenReceiver(to).onERC721Received(msg.sender, from, id, data) ==

```
https://github.com/code-423n4/2022-09-artgobblers/blob/d2087c5a8a6a4f1b9784520e7fe75afa3a9cbdbe/src/utils/token/PagesERC721.sol#L136

### [G&#x2011;04]  State variables should be cached in stack variables rather than re-reading them from storage
The instances below point to the second+ access of a state variable within a function. Caching of a state variable replaces each Gwarmaccess (**100 gas**) with a much cheaper stack read. Other less obvious fixes/optimizations include having local memory caches of state variable structs, or having local caches of state variable contracts/addresses.

*There are 4 instances of this issue:*
```solidity
File: src/ArtGobblers.sol

/// @audit BASE_URI on line 698
709:              return string.concat(BASE_URI, gobblerId.toString());

/// @audit numMintedFromGoo on line 482
493:              uint256 numMintedSinceStart = numMintedFromGoo - numMintedAtStart;

/// @audit getGobblerData[swapId].idx on line 615
617:                      : getGobblerData[swapId].idx; // Shuffled before.

/// @audit getGobblerData[currentId].idx on line 623
625:                      : getGobblerData[currentId].idx; // Shuffled before.

```
https://github.com/code-423n4/2022-09-artgobblers/blob/d2087c5a8a6a4f1b9784520e7fe75afa3a9cbdbe/src/ArtGobblers.sol#L709

### [G&#x2011;05]  Multiple accesses of a mapping/array should use a local variable cache
The instances below point to the second+ access of a value inside a mapping/array, within a function. Caching a mapping's value in a local `storage` or `calldata` variable when the value is accessed [multiple times](https://gist.github.com/IllIllI000/ec23a57daa30a8f8ca8b9681c8ccefb0), saves **~42 gas per access** due to not having to recalculate the key's keccak256 hash (Gkeccak256 - **30 gas**) and that calculation's associated stack operations. Caching an array's struct avoids recalculating the array offsets into memory/calldata

*There are 25 instances of this issue:*
```solidity
File: src/ArtGobblers.sol

/// @audit getGobblerData[id] on line 437
439:                  burnedMultipleTotal += getGobblerData[id].emissionMultiple;

/// @audit getGobblerData[id] on line 439
441:                  emit Transfer(msg.sender, getGobblerData[id].owner = address(0), id);

/// @audit getUserData[<etc>] on line 454
455:              getUserData[msg.sender].lastTimestamp = uint64(block.timestamp); // Store time alongside it.

/// @audit getUserData[<etc>] on line 455
456:              getUserData[msg.sender].emissionMultiple += uint32(burnedMultipleTotal); // Update multiple.

/// @audit getUserData[<etc>] on line 456
/// @audit getUserData[<etc>] on line 458
458:              getUserData[msg.sender].gobblersOwned = uint32(getUserData[msg.sender].gobblersOwned - cost + 1);

/// @audit getGobblerData[swapId] on line 615
617:                      : getGobblerData[swapId].idx; // Shuffled before.

/// @audit getGobblerData[currentId] on line 620
623:                  uint64 currentIndex = getGobblerData[currentId].idx == 0

/// @audit getGobblerData[currentId] on line 623
625:                      : getGobblerData[currentId].idx; // Shuffled before.

/// @audit getGobblerData[currentId] on line 625
649:                  getGobblerData[currentId].idx = swapIndex;

/// @audit getGobblerData[currentId] on line 649
650:                  getGobblerData[currentId].emissionMultiple = uint32(newCurrentIdMultiple);

/// @audit getGobblerData[swapId] on line 617
653:                  getGobblerData[swapId].idx = currentIndex;

/// @audit getUserData[currentIdOwner] on line 660
661:                  getUserData[currentIdOwner].lastTimestamp = uint64(block.timestamp);

/// @audit getUserData[currentIdOwner] on line 661
662:                  getUserData[currentIdOwner].emissionMultiple += uint32(newCurrentIdMultiple);

/// @audit getUserData[user] on line 761
762:              getUserData[user].lastBalance,

/// @audit getUserData[user] on line 762
763:              uint(toDaysWadUnsafe(block.timestamp - getUserData[user].lastTimestamp))

/// @audit getUserData[user] on line 825
826:          getUserData[user].lastTimestamp = uint64(block.timestamp);

/// @audit getGobblerData[id] on line 885
896:          getGobblerData[id].owner = to;

/// @audit getGobblerData[id] on line 896
899:              uint32 emissionMultiple = getGobblerData[id].emissionMultiple; // Caching saves gas.

/// @audit getUserData[from] on line 903
904:              getUserData[from].lastTimestamp = uint64(block.timestamp);

/// @audit getUserData[from] on line 904
905:              getUserData[from].emissionMultiple -= emissionMultiple;

/// @audit getUserData[from] on line 905
906:              getUserData[from].gobblersOwned -= 1;

/// @audit getUserData[to] on line 910
911:              getUserData[to].lastTimestamp = uint64(block.timestamp);

/// @audit getUserData[to] on line 911
912:              getUserData[to].emissionMultiple += emissionMultiple;

/// @audit getUserData[to] on line 912
913:              getUserData[to].gobblersOwned += 1;

```
https://github.com/code-423n4/2022-09-artgobblers/blob/d2087c5a8a6a4f1b9784520e7fe75afa3a9cbdbe/src/ArtGobblers.sol#L439

### [G&#x2011;06]  `<x> += <y>` costs more gas than `<x> = <x> + <y>` for state variables
Using the addition operator instead of plus-equals saves **[113 gas](https://gist.github.com/IllIllI000/cbbfb267425b898e5be734d4008d4fe8)**

*There are 2 instances of this issue:*
```solidity
File: src/ArtGobblers.sol

844:              uint256 newNumMintedForReserves = numMintedForReserves += (numGobblersEach << 1);

```
https://github.com/code-423n4/2022-09-artgobblers/blob/d2087c5a8a6a4f1b9784520e7fe75afa3a9cbdbe/src/ArtGobblers.sol#L844

```solidity
File: src/Pages.sol

244:              uint256 newNumMintedForCommunity = numMintedForCommunity += uint128(numPages);

```
https://github.com/code-423n4/2022-09-artgobblers/blob/d2087c5a8a6a4f1b9784520e7fe75afa3a9cbdbe/src/Pages.sol#L244

### [G&#x2011;07]  `<array>.length` should not be looked up in every loop of a `for`-loop
The overheads outlined below are _PER LOOP_, excluding the first loop
* storage arrays incur a Gwarmaccess (**100 gas**)
* memory arrays use `MLOAD` (**3 gas**)
* calldata arrays use `CALLDATALOAD` (**3 gas**)

Caching the length changes each of these to a `DUP<N>` (**3 gas**), and gets rid of the extra `DUP<N>` needed to store the stack offset

*There are 2 instances of this issue:*
```solidity
File: src/utils/token/GobblersERC1155B.sol

114:              for (uint256 i = 0; i < owners.length; ++i) {

```
https://github.com/code-423n4/2022-09-artgobblers/blob/d2087c5a8a6a4f1b9784520e7fe75afa3a9cbdbe/src/utils/token/GobblersERC1155B.sol#L114

```solidity
File: src/utils/GobblerReserve.sol

37:               for (uint256 i = 0; i < ids.length; i++) {

```
https://github.com/code-423n4/2022-09-artgobblers/blob/d2087c5a8a6a4f1b9784520e7fe75afa3a9cbdbe/src/utils/GobblerReserve.sol#L37

### [G&#x2011;08]  Optimize names to save gas
`public`/`external` function names and `public` member variable names can be optimized to save gas. See [this](https://gist.github.com/IllIllI000/a5d8b486a8259f9f77891a919febd1a9) link for an example of how it works. Below are the interfaces/abstract contracts that can be optimized so that the most frequently-called functions use the least amount of gas possible during method lookup. Method IDs that have two leading zero bytes can save **128 gas** each during deployment, and renaming functions to have lower method IDs will save **22 gas** per call, [per sorted position shifted](https://medium.com/joyso/solidity-how-does-function-name-affect-gas-consumption-in-smart-contract-47d270d8ac92)

*There are 10 instances of this issue:*
```solidity
File: src/ArtGobblers.sol

/// @audit claimGobbler(), mintFromGoo(), gobblerPrice(), mintLegendaryGobbler(), legendaryGobblerPrice(), requestRandomSeed(), acceptRandomSeed(), upgradeRandProvider(), revealGobblers(), gobble(), gooBalance(), addGoo(), removeGoo(), burnGooForPages(), mintReservedGobblers(), getGobblerEmissionMultiple(), getUserEmissionMultiple()
83:   contract ArtGobblers is GobblersERC721, LogisticVRGDA, Owned, ERC1155TokenReceiver {

```
https://github.com/code-423n4/2022-09-artgobblers/blob/d2087c5a8a6a4f1b9784520e7fe75afa3a9cbdbe/src/ArtGobblers.sol#L83

```solidity
File: script/deploy/DeployBase.s.sol

/// @audit run()
16:   abstract contract DeployBase is Script {

```
https://github.com/code-423n4/2022-09-artgobblers/blob/d2087c5a8a6a4f1b9784520e7fe75afa3a9cbdbe/script/deploy/DeployBase.s.sol#L16

```solidity
File: src/Pages.sol

/// @audit mintFromGoo(), pagePrice(), mintCommunityPages()
78:   contract Pages is PagesERC721, LogisticToLinearVRGDA {

```
https://github.com/code-423n4/2022-09-artgobblers/blob/d2087c5a8a6a4f1b9784520e7fe75afa3a9cbdbe/src/Pages.sol#L78

```solidity
File: src/utils/rand/ChainlinkV1RandProvider.sol

/// @audit requestRandomBytes()
14:   contract ChainlinkV1RandProvider is RandProvider, VRFConsumerBase {

```
https://github.com/code-423n4/2022-09-artgobblers/blob/d2087c5a8a6a4f1b9784520e7fe75afa3a9cbdbe/src/utils/rand/ChainlinkV1RandProvider.sol#L14

```solidity
File: src/Goo.sol

/// @audit mintForGobblers(), burnForGobblers(), burnForPages()
58:   contract Goo is ERC20("Goo", "GOO", 18) {

```
https://github.com/code-423n4/2022-09-artgobblers/blob/d2087c5a8a6a4f1b9784520e7fe75afa3a9cbdbe/src/Goo.sol#L58

```solidity
File: lib/VRGDAs/src/VRGDA.sol

/// @audit getVRGDAPrice(), getTargetSaleTime()
10:   abstract contract VRGDA {

```
https://github.com/transmissions11/VRGDAs/blob/f4dec0611641e344339b2c78e5f733bba3b532e0/src/VRGDA.sol#L10

```solidity
File: lib/goo-issuance/src/LibGOO.sol

/// @audit computeGOOBalance()
10:   library LibGOO {

```
https://github.com/transmissions11/goo-issuance/blob/648e65e66e43ff5c19681427e300ece9c0df1437/src/LibGOO.sol#L10

```solidity
File: lib/solmate/src/auth/Owned.sol

/// @audit setOwner()
6:    abstract contract Owned {

```
https://github.com/transmissions11/solmate/blob/26572802743101f160f2d07556edfc162896115e/src/auth/Owned.sol#L6

```solidity
File: src/utils/GobblerReserve.sol

/// @audit withdraw()
12:   contract GobblerReserve is Owned {

```
https://github.com/code-423n4/2022-09-artgobblers/blob/d2087c5a8a6a4f1b9784520e7fe75afa3a9cbdbe/src/utils/GobblerReserve.sol#L12

```solidity
File: src/utils/rand/RandProvider.sol

/// @audit requestRandomBytes()
8:    interface RandProvider {

```
https://github.com/code-423n4/2022-09-artgobblers/blob/d2087c5a8a6a4f1b9784520e7fe75afa3a9cbdbe/src/utils/rand/RandProvider.sol#L8

### [G&#x2011;09]  Using `bool`s for storage incurs overhead
```solidity
    // Booleans are more expensive than uint256 or any type that takes up a full
    // word because each write operation emits an extra SLOAD to first read the
    // slot's contents, replace the bits taken up by the boolean, and then write
    // back. This is the compiler's defense against contract upgrades and
    // pointer aliasing, and it cannot be disabled.
```
https://github.com/OpenZeppelin/openzeppelin-contracts/blob/58f635312aa21f947cae5f8578638a85aa2519f5/contracts/security/ReentrancyGuard.sol#L23-L27
Use `uint256(1)` and `uint256(2)` for true/false to avoid a Gwarmaccess (**[100 gas](https://gist.github.com/IllIllI000/1b70014db712f8572a72378321250058)**) for the extra SLOAD, and to avoid Gsset (**20000 gas**) when changing from `false` to `true`, after having been `true` in the past

*There are 5 instances of this issue:*
```solidity
File: src/ArtGobblers.sol

149:      mapping(address => bool) public hasClaimedMintlistGobbler;

```
https://github.com/code-423n4/2022-09-artgobblers/blob/d2087c5a8a6a4f1b9784520e7fe75afa3a9cbdbe/src/ArtGobblers.sol#L149

```solidity
File: lib/solmate/src/tokens/ERC721.sol

51:       mapping(address => mapping(address => bool)) public isApprovedForAll;

```
https://github.com/transmissions11/solmate/blob/26572802743101f160f2d07556edfc162896115e/src/tokens/ERC721.sol#L51

```solidity
File: src/utils/token/GobblersERC1155B.sol

37:       mapping(address => mapping(address => bool)) public isApprovedForAll;

```
https://github.com/code-423n4/2022-09-artgobblers/blob/d2087c5a8a6a4f1b9784520e7fe75afa3a9cbdbe/src/utils/token/GobblersERC1155B.sol#L37

```solidity
File: src/utils/token/GobblersERC721.sol

77:       mapping(address => mapping(address => bool)) public isApprovedForAll;

```
https://github.com/code-423n4/2022-09-artgobblers/blob/d2087c5a8a6a4f1b9784520e7fe75afa3a9cbdbe/src/utils/token/GobblersERC721.sol#L77

```solidity
File: src/utils/token/PagesERC721.sol

70:       mapping(address => mapping(address => bool)) internal _isApprovedForAll;

```
https://github.com/code-423n4/2022-09-artgobblers/blob/d2087c5a8a6a4f1b9784520e7fe75afa3a9cbdbe/src/utils/token/PagesERC721.sol#L70

### [G&#x2011;10]  Use a more recent version of solidity
Use a solidity version of at least 0.8.2 to get simple compiler automatic inlining
Use a solidity version of at least 0.8.3 to get better struct packing and cheaper multiple storage reads
Use a solidity version of at least 0.8.4 to get custom errors, which are cheaper at deployment than `revert()/require()` strings
Use a solidity version of at least 0.8.10 to have external calls skip contract existence checks if the external call has a return value

*There are 22 instances of this issue:*
```solidity
File: src/ArtGobblers.sol

2:    pragma solidity >=0.8.0;

```
https://github.com/code-423n4/2022-09-artgobblers/blob/d2087c5a8a6a4f1b9784520e7fe75afa3a9cbdbe/src/ArtGobblers.sol#L2

```solidity
File: lib/solmate/src/utils/FixedPointMathLib.sol

2:    pragma solidity >=0.8.0;

```
https://github.com/transmissions11/solmate/blob/26572802743101f160f2d07556edfc162896115e/src/utils/FixedPointMathLib.sol#L2

```solidity
File: lib/solmate/src/tokens/ERC721.sol

2:    pragma solidity >=0.8.0;

```
https://github.com/transmissions11/solmate/blob/26572802743101f160f2d07556edfc162896115e/src/tokens/ERC721.sol#L2

```solidity
File: src/utils/token/GobblersERC1155B.sol

2:    pragma solidity >=0.8.0;

```
https://github.com/code-423n4/2022-09-artgobblers/blob/d2087c5a8a6a4f1b9784520e7fe75afa3a9cbdbe/src/utils/token/GobblersERC1155B.sol#L2

```solidity
File: lib/solmate/src/utils/SignedWadMath.sol

2:    pragma solidity >=0.8.0;

```
https://github.com/transmissions11/solmate/blob/26572802743101f160f2d07556edfc162896115e/src/utils/SignedWadMath.sol#L2

```solidity
File: src/utils/token/GobblersERC721.sol

2:    pragma solidity >=0.8.0;

```
https://github.com/code-423n4/2022-09-artgobblers/blob/d2087c5a8a6a4f1b9784520e7fe75afa3a9cbdbe/src/utils/token/GobblersERC721.sol#L2

```solidity
File: src/utils/token/PagesERC721.sol

2:    pragma solidity >=0.8.0;

```
https://github.com/code-423n4/2022-09-artgobblers/blob/d2087c5a8a6a4f1b9784520e7fe75afa3a9cbdbe/src/utils/token/PagesERC721.sol#L2

```solidity
File: script/deploy/DeployBase.s.sol

2:    pragma solidity >=0.8.0;

```
https://github.com/code-423n4/2022-09-artgobblers/blob/d2087c5a8a6a4f1b9784520e7fe75afa3a9cbdbe/script/deploy/DeployBase.s.sol#L2

```solidity
File: src/Pages.sol

2:    pragma solidity >=0.8.0;

```
https://github.com/code-423n4/2022-09-artgobblers/blob/d2087c5a8a6a4f1b9784520e7fe75afa3a9cbdbe/src/Pages.sol#L2

```solidity
File: src/utils/rand/ChainlinkV1RandProvider.sol

2:    pragma solidity >=0.8.0;

```
https://github.com/code-423n4/2022-09-artgobblers/blob/d2087c5a8a6a4f1b9784520e7fe75afa3a9cbdbe/src/utils/rand/ChainlinkV1RandProvider.sol#L2

```solidity
File: lib/VRGDAs/src/LogisticToLinearVRGDA.sol

2:    pragma solidity >=0.8.0;

```
https://github.com/transmissions11/VRGDAs/blob/f4dec0611641e344339b2c78e5f733bba3b532e0/src/LogisticToLinearVRGDA.sol#L2

```solidity
File: lib/solmate/src/utils/MerkleProofLib.sol

2:    pragma solidity >=0.8.0;

```
https://github.com/transmissions11/solmate/blob/26572802743101f160f2d07556edfc162896115e/src/utils/MerkleProofLib.sol#L2

```solidity
File: script/deploy/DeployRinkeby.s.sol

2:    pragma solidity >=0.8.0;

```
https://github.com/code-423n4/2022-09-artgobblers/blob/d2087c5a8a6a4f1b9784520e7fe75afa3a9cbdbe/script/deploy/DeployRinkeby.s.sol#L2

```solidity
File: src/Goo.sol

2:    pragma solidity >=0.8.0;

```
https://github.com/code-423n4/2022-09-artgobblers/blob/d2087c5a8a6a4f1b9784520e7fe75afa3a9cbdbe/src/Goo.sol#L2

```solidity
File: lib/VRGDAs/src/LogisticVRGDA.sol

2:    pragma solidity >=0.8.0;

```
https://github.com/transmissions11/VRGDAs/blob/f4dec0611641e344339b2c78e5f733bba3b532e0/src/LogisticVRGDA.sol#L2

```solidity
File: lib/solmate/src/utils/LibString.sol

2:    pragma solidity >=0.8.0;

```
https://github.com/transmissions11/solmate/blob/26572802743101f160f2d07556edfc162896115e/src/utils/LibString.sol#L2

```solidity
File: lib/VRGDAs/src/VRGDA.sol

2:    pragma solidity >=0.8.0;

```
https://github.com/transmissions11/VRGDAs/blob/f4dec0611641e344339b2c78e5f733bba3b532e0/src/VRGDA.sol#L2

```solidity
File: lib/goo-issuance/src/LibGOO.sol

2:    pragma solidity >=0.8.0;

```
https://github.com/transmissions11/goo-issuance/blob/648e65e66e43ff5c19681427e300ece9c0df1437/src/LibGOO.sol#L2

```solidity
File: lib/solmate/src/auth/Owned.sol

2:    pragma solidity >=0.8.0;

```
https://github.com/transmissions11/solmate/blob/26572802743101f160f2d07556edfc162896115e/src/auth/Owned.sol#L2

```solidity
File: lib/VRGDAs/src/LinearVRGDA.sol

2:    pragma solidity >=0.8.0;

```
https://github.com/transmissions11/VRGDAs/blob/f4dec0611641e344339b2c78e5f733bba3b532e0/src/LinearVRGDA.sol#L2

```solidity
File: src/utils/GobblerReserve.sol

2:    pragma solidity >=0.8.0;

```
https://github.com/code-423n4/2022-09-artgobblers/blob/d2087c5a8a6a4f1b9784520e7fe75afa3a9cbdbe/src/utils/GobblerReserve.sol#L2

```solidity
File: src/utils/rand/RandProvider.sol

2:    pragma solidity >=0.8.0;

```
https://github.com/code-423n4/2022-09-artgobblers/blob/d2087c5a8a6a4f1b9784520e7fe75afa3a9cbdbe/src/utils/rand/RandProvider.sol#L2

### [G&#x2011;11]  `++i` costs less gas than `i++`, especially when it's used in `for`-loops (`--i`/`i--` too)
Saves **5 gas per loop**

*There are 9 instances of this issue:*
```solidity
File: lib/solmate/src/tokens/ERC721.sol

99:               _balanceOf[from]--;

101:              _balanceOf[to]++;

164:              _balanceOf[to]++;

179:              _balanceOf[owner]--;

```
https://github.com/transmissions11/solmate/blob/26572802743101f160f2d07556edfc162896115e/src/tokens/ERC721.sol#L99

```solidity
File: src/utils/token/PagesERC721.sol

115:              _balanceOf[from]--;

117:              _balanceOf[to]++;

181:              _balanceOf[to]++;

```
https://github.com/code-423n4/2022-09-artgobblers/blob/d2087c5a8a6a4f1b9784520e7fe75afa3a9cbdbe/src/utils/token/PagesERC721.sol#L115

```solidity
File: src/Pages.sol

251:              for (uint256 i = 0; i < numPages; i++) _mint(community, ++lastMintedPageId);

```
https://github.com/code-423n4/2022-09-artgobblers/blob/d2087c5a8a6a4f1b9784520e7fe75afa3a9cbdbe/src/Pages.sol#L251

```solidity
File: src/utils/GobblerReserve.sol

37:               for (uint256 i = 0; i < ids.length; i++) {

```
https://github.com/code-423n4/2022-09-artgobblers/blob/d2087c5a8a6a4f1b9784520e7fe75afa3a9cbdbe/src/utils/GobblerReserve.sol#L37

### [G&#x2011;12]  Using `private` rather than `public` for constants, saves gas
If needed, the values can be read from the verified contract source code, or if there are multiple values there can be a single getter function that [returns a tuple](https://github.com/code-423n4/2022-08-frax/blob/90f55a9ce4e25bceed3a74290b854341d8de6afa/src/contracts/FraxlendPair.sol#L156-L178) of the values of all currently-public constants. Saves **3406-3606 gas** in deployment gas due to the compiler not having to create non-payable getter functions for deployment calldata, not having to store the bytes of the value outside of where it's used, and not adding another entry to the method ID table

*There are 18 instances of this issue:*
```solidity
File: src/ArtGobblers.sol

112:      uint256 public constant MAX_SUPPLY = 10000;

115:      uint256 public constant MINTLIST_SUPPLY = 2000;

118:      uint256 public constant LEGENDARY_SUPPLY = 10;

122:      uint256 public constant RESERVED_SUPPLY = (MAX_SUPPLY - MINTLIST_SUPPLY - LEGENDARY_SUPPLY) / 5;

126       uint256 public constant MAX_MINTABLE = MAX_SUPPLY
127           - MINTLIST_SUPPLY
128           - LEGENDARY_SUPPLY
129:          - RESERVED_SUPPLY;

146:      bytes32 public immutable merkleRoot;

156:      uint256 public immutable mintStart;

177:      uint256 public constant LEGENDARY_GOBBLER_INITIAL_START_PRICE = 69;

180:      uint256 public constant FIRST_LEGENDARY_GOBBLER_ID = MAX_SUPPLY - LEGENDARY_SUPPLY + 1;

184:      uint256 public constant LEGENDARY_AUCTION_INTERVAL = MAX_MINTABLE / (LEGENDARY_SUPPLY + 1);

```
https://github.com/code-423n4/2022-09-artgobblers/blob/d2087c5a8a6a4f1b9784520e7fe75afa3a9cbdbe/src/ArtGobblers.sol#L112

```solidity
File: src/Pages.sol

103:      uint256 public immutable mintStart;

```
https://github.com/code-423n4/2022-09-artgobblers/blob/d2087c5a8a6a4f1b9784520e7fe75afa3a9cbdbe/src/Pages.sol#L103

```solidity
File: script/deploy/DeployRinkeby.s.sol

11:       uint256 public immutable mintStart = 1656369768;

13:       string public constant gobblerBaseUri = "https://testnet.ag.xyz/api/nfts/gobblers/";

14:       string public constant gobblerUnrevealedUri = "https://testnet.ag.xyz/api/nfts/unrevealed";

15:       string public constant pagesBaseUri = "https://testnet.ag.xyz/api/nfts/pages/";

```
https://github.com/code-423n4/2022-09-artgobblers/blob/d2087c5a8a6a4f1b9784520e7fe75afa3a9cbdbe/script/deploy/DeployRinkeby.s.sol#L11

```solidity
File: lib/VRGDAs/src/LogisticVRGDA.sol

20:       int256 public immutable logisticLimit;

25:       int256 public immutable logisticLimitDoubled;

```
https://github.com/transmissions11/VRGDAs/blob/f4dec0611641e344339b2c78e5f733bba3b532e0/src/LogisticVRGDA.sol#L20

```solidity
File: lib/VRGDAs/src/VRGDA.sol

17:       int256 public immutable targetPrice;

```
https://github.com/transmissions11/VRGDAs/blob/f4dec0611641e344339b2c78e5f733bba3b532e0/src/VRGDA.sol#L17

### [G&#x2011;13]  Division by two should use bit shifting
`<x> / 2` is the same as `<x> >> 1`. While the compiler uses the `SHR` opcode to accomplish both, the version that uses division incurs an overhead of [**20 gas**](https://gist.github.com/IllIllI000/ec0e4e6c4f52a6bca158f137a3afd4ff) due to `JUMP`s to and from a compiler utility function that introduces checks which can be avoided by using `unchecked {}` around the division by two

*There is 1 instance of this issue:*
```solidity
File: src/ArtGobblers.sol

462:                  cost <= LEGENDARY_GOBBLER_INITIAL_START_PRICE / 2 ? LEGENDARY_GOBBLER_INITIAL_START_PRICE : cost * 2

```
https://github.com/code-423n4/2022-09-artgobblers/blob/d2087c5a8a6a4f1b9784520e7fe75afa3a9cbdbe/src/ArtGobblers.sol#L462

### [G&#x2011;14]  Stack variable used as a cheaper cache for a state variable is only used once
If the variable is only accessed once, it's cheaper to use the state variable directly that one time, and save the **3 gas** the extra stack assignment would spend

*There are 2 instances of this issue:*
```solidity
File: src/ArtGobblers.sol

480:          uint256 startPrice = legendaryGobblerAuctionData.startPrice;

481:          uint256 numSold = legendaryGobblerAuctionData.numSold;

```
https://github.com/code-423n4/2022-09-artgobblers/blob/d2087c5a8a6a4f1b9784520e7fe75afa3a9cbdbe/src/ArtGobblers.sol#L480

### [G&#x2011;15]  Use custom errors rather than `revert()`/`require()` strings to save gas
Custom errors are available from solidity version 0.8.4. Custom errors save [**~50 gas**](https://gist.github.com/IllIllI000/ad1bd0d29a0101b25e57c293b4b0c746) each time they're hit by [avoiding having to allocate and store the revert string](https://blog.soliditylang.org/2021/04/21/custom-errors/#errors-in-depth). Not defining the strings also save deployment gas

*There are 36 instances of this issue:*
```solidity
File: src/ArtGobblers.sol

437:                  require(getGobblerData[id].owner == msg.sender, "WRONG_FROM");

885:          require(from == getGobblerData[id].owner, "WRONG_FROM");

887:          require(to != address(0), "INVALID_RECIPIENT");

889           require(
890               msg.sender == from || isApprovedForAll[from][msg.sender] || msg.sender == getApproved[id],
891               "NOT_AUTHORIZED"
892:          );

```
https://github.com/code-423n4/2022-09-artgobblers/blob/d2087c5a8a6a4f1b9784520e7fe75afa3a9cbdbe/src/ArtGobblers.sol#L437

```solidity
File: lib/solmate/src/tokens/ERC721.sol

36:           require((owner = _ownerOf[id]) != address(0), "NOT_MINTED");

40:           require(owner != address(0), "ZERO_ADDRESS");

69:           require(msg.sender == owner || isApprovedForAll[owner][msg.sender], "NOT_AUTHORIZED");

87:           require(from == _ownerOf[id], "WRONG_FROM");

89:           require(to != address(0), "INVALID_RECIPIENT");

91            require(
92                msg.sender == from || isApprovedForAll[from][msg.sender] || msg.sender == getApproved[id],
93                "NOT_AUTHORIZED"
94:           );

118           require(
119               to.code.length == 0 ||
120                   ERC721TokenReceiver(to).onERC721Received(msg.sender, from, id, "") ==
121                   ERC721TokenReceiver.onERC721Received.selector,
122               "UNSAFE_RECIPIENT"
123:          );

134           require(
135               to.code.length == 0 ||
136                   ERC721TokenReceiver(to).onERC721Received(msg.sender, from, id, data) ==
137                   ERC721TokenReceiver.onERC721Received.selector,
138               "UNSAFE_RECIPIENT"
139:          );

158:          require(to != address(0), "INVALID_RECIPIENT");

160:          require(_ownerOf[id] == address(0), "ALREADY_MINTED");

175:          require(owner != address(0), "NOT_MINTED");

196           require(
197               to.code.length == 0 ||
198                   ERC721TokenReceiver(to).onERC721Received(msg.sender, address(0), id, "") ==
199                   ERC721TokenReceiver.onERC721Received.selector,
200               "UNSAFE_RECIPIENT"
201:          );

211           require(
212               to.code.length == 0 ||
213                   ERC721TokenReceiver(to).onERC721Received(msg.sender, address(0), id, data) ==
214                   ERC721TokenReceiver.onERC721Received.selector,
215               "UNSAFE_RECIPIENT"
216:          );

```
https://github.com/transmissions11/solmate/blob/26572802743101f160f2d07556edfc162896115e/src/tokens/ERC721.sol#L36

```solidity
File: src/utils/token/GobblersERC1155B.sol

107:          require(owners.length == ids.length, "LENGTH_MISMATCH");

149               require(
150                   ERC1155TokenReceiver(to).onERC1155Received(msg.sender, address(0), id, 1, data) ==
151                       ERC1155TokenReceiver.onERC1155Received.selector,
152                   "UNSAFE_RECIPIENT"
153:              );

185               require(
186                   ERC1155TokenReceiver(to).onERC1155BatchReceived(msg.sender, address(0), ids, amounts, data) ==
187                       ERC1155TokenReceiver.onERC1155BatchReceived.selector,
188                   "UNSAFE_RECIPIENT"
189:              );

```
https://github.com/code-423n4/2022-09-artgobblers/blob/d2087c5a8a6a4f1b9784520e7fe75afa3a9cbdbe/src/utils/token/GobblersERC1155B.sol#L107

```solidity
File: lib/solmate/src/utils/SignedWadMath.sol

142:          require(x > 0, "UNDEFINED");

```
https://github.com/transmissions11/solmate/blob/26572802743101f160f2d07556edfc162896115e/src/utils/SignedWadMath.sol#L142

```solidity
File: src/utils/token/GobblersERC721.sol

62:           require((owner = getGobblerData[id].owner) != address(0), "NOT_MINTED");

66:           require(owner != address(0), "ZERO_ADDRESS");

95:           require(msg.sender == owner || isApprovedForAll[owner][msg.sender], "NOT_AUTHORIZED");

121           require(
122               to.code.length == 0 ||
123                   ERC721TokenReceiver(to).onERC721Received(msg.sender, from, id, "") ==
124                   ERC721TokenReceiver.onERC721Received.selector,
125               "UNSAFE_RECIPIENT"
126:          );

137           require(
138               to.code.length == 0 ||
139                   ERC721TokenReceiver(to).onERC721Received(msg.sender, from, id, data) ==
140                   ERC721TokenReceiver.onERC721Received.selector,
141               "UNSAFE_RECIPIENT"
142:          );

```
https://github.com/code-423n4/2022-09-artgobblers/blob/d2087c5a8a6a4f1b9784520e7fe75afa3a9cbdbe/src/utils/token/GobblersERC721.sol#L62

```solidity
File: src/utils/token/PagesERC721.sol

55:           require((owner = _ownerOf[id]) != address(0), "NOT_MINTED");

59:           require(owner != address(0), "ZERO_ADDRESS");

85:           require(msg.sender == owner || isApprovedForAll(owner, msg.sender), "NOT_AUTHORIZED");

103:          require(from == _ownerOf[id], "WRONG_FROM");

105:          require(to != address(0), "INVALID_RECIPIENT");

107           require(
108               msg.sender == from || isApprovedForAll(from, msg.sender) || msg.sender == getApproved[id],
109               "NOT_AUTHORIZED"
110:          );

135               require(
136                   ERC721TokenReceiver(to).onERC721Received(msg.sender, from, id, "") ==
137                       ERC721TokenReceiver.onERC721Received.selector,
138                   "UNSAFE_RECIPIENT"
139:              );

151               require(
152                   ERC721TokenReceiver(to).onERC721Received(msg.sender, from, id, data) ==
153                       ERC721TokenReceiver.onERC721Received.selector,
154                   "UNSAFE_RECIPIENT"
155:              );

```
https://github.com/code-423n4/2022-09-artgobblers/blob/d2087c5a8a6a4f1b9784520e7fe75afa3a9cbdbe/src/utils/token/PagesERC721.sol#L55

```solidity
File: lib/VRGDAs/src/VRGDA.sol

32:           require(decayConstant < 0, "NON_NEGATIVE_DECAY_CONSTANT");

```
https://github.com/transmissions11/VRGDAs/blob/f4dec0611641e344339b2c78e5f733bba3b532e0/src/VRGDA.sol#L32

```solidity
File: lib/solmate/src/auth/Owned.sol

20:           require(msg.sender == owner, "UNAUTHORIZED");

```
https://github.com/transmissions11/solmate/blob/26572802743101f160f2d07556edfc162896115e/src/auth/Owned.sol#L20

### [G&#x2011;16]  Functions guaranteed to revert when called by normal users can be marked `payable`
If a function modifier such as `onlyOwner` is used, the function will revert if a normal user tries to pay the function. Marking the function as `payable` will lower the gas cost for legitimate callers because the compiler will not include checks for whether a payment was provided. The extra opcodes avoided are 
`CALLVALUE`(2),`DUP1`(3),`ISZERO`(3),`PUSH2`(3),`JUMPI`(10),`PUSH1`(3),`DUP1`(3),`REVERT`(0),`JUMPDEST`(1),`POP`(2), which costs an average of about **21 gas per call** to the function, in addition to the extra deployment cost

*There are 3 instances of this issue:*
```solidity
File: src/ArtGobblers.sol

560:      function upgradeRandProvider(RandProvider newRandProvider) external onlyOwner {

```
https://github.com/code-423n4/2022-09-artgobblers/blob/d2087c5a8a6a4f1b9784520e7fe75afa3a9cbdbe/src/ArtGobblers.sol#L560

```solidity
File: lib/solmate/src/auth/Owned.sol

39:       function setOwner(address newOwner) public virtual onlyOwner {

```
https://github.com/transmissions11/solmate/blob/26572802743101f160f2d07556edfc162896115e/src/auth/Owned.sol#L39

```solidity
File: src/utils/GobblerReserve.sol

34:       function withdraw(address to, uint256[] calldata ids) external onlyOwner {

```
https://github.com/code-423n4/2022-09-artgobblers/blob/d2087c5a8a6a4f1b9784520e7fe75afa3a9cbdbe/src/utils/GobblerReserve.sol#L34


