## Tags

- bug
- QA (Quality Assurance)
- old-submission-method
- selected for report

# [QA Report](https://github.com/code-423n4/2022-09-artgobblers-findings/issues/323) 

## Summary

### Low Risk Issues
| |Issue|Instances|
|-|:-|:-:|
| [L&#x2011;01] | Don't roll your own crypto | 1 |
| [L&#x2011;02] | Chainlink's VRF V1 is deprecated | 1 |
| [L&#x2011;03] | Rinkbe is not supported for Chainlink's VRF | 1 |
| [L&#x2011;04] | Missing checks for `address(0x0)` when assigning values to `address` state variables | 10 |

Total: 13 instances over 4 issues

### Non-critical Issues
| |Issue|Instances|
|-|:-|:-:|
| [N&#x2011;01] | `requestId` is always zero | 1 |
| [N&#x2011;02] | Don't use periods with fragments | 1 |
| [N&#x2011;03] | Contract implements interface without extending the interface | 1 |
| [N&#x2011;04] | `public` functions not called by the contract should be declared `external` instead | 1 |
| [N&#x2011;05] | `constant`s should be defined rather than using magic numbers | 112 |
| [N&#x2011;06] | Use a more recent version of solidity | 2 |
| [N&#x2011;07] | Use a more recent version of solidity | 2 |
| [N&#x2011;08] | Constant redefined elsewhere | 8 |
| [N&#x2011;09] | Variable names that consist of all capital letters should be reserved for `constant`/`immutable` variables | 3 |
| [N&#x2011;10] | File is missing NatSpec | 2 |
| [N&#x2011;11] | NatSpec is incomplete | 10 |
| [N&#x2011;12] | Event is missing `indexed` fields | 16 |
| [N&#x2011;13] | Not using the named return variables anywhere in the function is confusing | 3 |
| [N&#x2011;14] | Duplicated `require()`/`revert()` checks should be refactored to a modifier or function | 2 |

Total: 164 instances over 14 issues




## Low Risk Issues

### [L&#x2011;01]  Don't roll your own crypto
The general advice when it comes to cryptography is [don't roll your own crypto](https://security.stackexchange.com/a/18198). The Chainlink VRF best practices page says that in order to get multiple values from a single random value, one should [hash a counter along with the random value](https://docs.chain.link/docs/vrf/v1/best-practices/#getting-multiple-random-numbers). This project does not follow that guidance, and instead recursively hashes previous hashes. Logically, if you have a source of entropy, then truncate, and hash again, over and over, you're going to lose entropy if there's a weakness in the hashing function, and every hashing function will eventually be found to have a weakness. Unless there is a very good reason not to, the code should follow the best practice Chainlink outlines. Furthermore, the current scheme wastes gas updating the random seed in every function call, as is outlined in my separate gas report.

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
675                  }
676              }
677  
678              // Update all relevant reveal state.
679:             gobblerRevealsData.randomSeed = uint64(randomSeed);

```
https://github.com/code-423n4/2022-09-artgobblers/blob/d2087c5a8a6a4f1b9784520e7fe75afa3a9cbdbe/src/ArtGobblers.sol#L664-L679

### [L&#x2011;02]  Chainlink's VRF V1 is deprecated
VRF V1 is [deprecated](https://docs.chain.link/docs/vrf/v1/introduction/), so new projects should not use it

*There is 1 instance of this issue:*
```solidity
File: /src/utils/rand/ChainlinkV1RandProvider.sol

13   /// @notice RandProvider wrapper around Chainlink VRF v1.
14:  contract ChainlinkV1RandProvider is RandProvider, VRFConsumerBase {

```
https://github.com/code-423n4/2022-09-artgobblers/blob/d2087c5a8a6a4f1b9784520e7fe75afa3a9cbdbe/src/utils/rand/ChainlinkV1RandProvider.sol#L13-L14

### [L&#x2011;03]  Rinkbe is not supported for Chainlink's VRF
Rinkeby is deprecated and not listed as supported for [Chainlink](https://docs.chain.link/docs/vrf/v1/supported-networks/). The documentation states `Once the Ethereum mainnet transitions to proof-of-stake, Rinkeby will no longer be an accurate staging environment for mainnet. ... Developers who currently use Rinkeby as a staging/testing environment should prioritize migrating to Goerli or Sepolia` [link](https://blog.ethereum.org/2022/06/21/testnet-deprecation)

*There is 1 instance of this issue:*
```solidity
File: /script/deploy/DeployRinkeby.s.sol

6:   contract DeployRinkeby is DeployBase {

```
https://github.com/code-423n4/2022-09-artgobblers/blob/d2087c5a8a6a4f1b9784520e7fe75afa3a9cbdbe/script/deploy/DeployRinkeby.s.sol#L6

### [L&#x2011;04]  Missing checks for `address(0x0)` when assigning values to `address` state variables

*There are 10 instances of this issue:*
```solidity
File: src/ArtGobblers.sol

316:          team = _team;

317:          community = _community;

```
https://github.com/code-423n4/2022-09-artgobblers/blob/d2087c5a8a6a4f1b9784520e7fe75afa3a9cbdbe/src/ArtGobblers.sol#L316

```solidity
File: script/deploy/DeployBase.s.sol

49:           teamColdWallet = _teamColdWallet;

52:           vrfCoordinator = _vrfCoordinator;

53:           linkToken = _linkToken;

```
https://github.com/code-423n4/2022-09-artgobblers/blob/d2087c5a8a6a4f1b9784520e7fe75afa3a9cbdbe/script/deploy/DeployBase.s.sol#L49

```solidity
File: src/Pages.sol

181:          community = _community;

```
https://github.com/code-423n4/2022-09-artgobblers/blob/d2087c5a8a6a4f1b9784520e7fe75afa3a9cbdbe/src/Pages.sol#L181

```solidity
File: src/Goo.sol

83:           artGobblers = _artGobblers;

84:           pages = _pages;

```
https://github.com/code-423n4/2022-09-artgobblers/blob/d2087c5a8a6a4f1b9784520e7fe75afa3a9cbdbe/src/Goo.sol#L83

```solidity
File: lib/solmate/src/auth/Owned.sol

30:           owner = _owner;

40:           owner = newOwner;

```
https://github.com/transmissions11/solmate/blob/26572802743101f160f2d07556edfc162896115e/src/auth/Owned.sol#L30

## Non-critical Issues

### [N&#x2011;01]  `requestId` is always zero
Chainlink suggests to have a unique `requestId` for every separate randomness request. By always using the same value, it's not possible to tell whether Chainlink returned data for a valid request, or if there was some Chainlink bug that triggered a callback for a request that was never made

*There is 1 instance of this issue:*
```solidity
File: /src/utils/rand/ChainlinkV1RandProvider.sol

62       function requestRandomBytes() external returns (bytes32 requestId) {
63           // The caller must be the ArtGobblers contract, revert otherwise.
64           if (msg.sender != address(artGobblers)) revert NotGobblers();
65   
66:          emit RandomBytesRequested(requestId);

```
https://github.com/code-423n4/2022-09-artgobblers/blob/d2087c5a8a6a4f1b9784520e7fe75afa3a9cbdbe/src/utils/rand/ChainlinkV1RandProvider.sol#L62-L66

### [N&#x2011;02]  Don't use periods with fragments
Throughout the files, most of the comments have fragments that end with periods. They should either be converted to actual sentences with both a noun phrase and a verb phrase, or the periods should be removed.

*There is 1 instance of this issue:*
```solidity
File: /src/ArtGobblers.sol


```
https://github.com/code-423n4/2022-09-artgobblers/blob/d2087c5a8a6a4f1b9784520e7fe75afa3a9cbdbe/src/ArtGobblers.sol

### [N&#x2011;03]  Contract implements interface without extending the interface
Not extending the interface may lead to the wrong function signature being used, leading to unexpected behavior. If the interface is in fact being implemented, use the `override` keyword to indicate that fact

*There is 1 instance of this issue:*
```solidity
File: src/utils/token/GobblersERC1155B.sol

/// @audit IERC1155MetadataURI.balanceOf(), IERC1155MetadataURI.uri(), IERC1155MetadataURI.safeTransferFrom(), IERC1155MetadataURI.safeBatchTransferFrom(), IERC1155MetadataURI.balanceOfBatch()
8:    abstract contract GobblersERC1155B {

```
https://github.com/code-423n4/2022-09-artgobblers/blob/d2087c5a8a6a4f1b9784520e7fe75afa3a9cbdbe/src/utils/token/GobblersERC1155B.sol#L8

### [N&#x2011;04]  `public` functions not called by the contract should be declared `external` instead
Contracts [are allowed](https://docs.soliditylang.org/en/latest/contracts.html#function-overriding) to override their parents' functions and change the visibility from `external` to `public`.

*There is 1 instance of this issue:*
```solidity
File: lib/goo-issuance/src/LibGOO.sol

17        function computeGOOBalance(
18            uint256 emissionMultiple,
19            uint256 lastBalanceWad,
20            uint256 timeElapsedWad
21:       ) public pure returns (uint256) {

```
https://github.com/transmissions11/goo-issuance/blob/648e65e66e43ff5c19681427e300ece9c0df1437/src/LibGOO.sol#L17-L21

### [N&#x2011;05]  `constant`s should be defined rather than using magic numbers
Even [assembly](https://github.com/code-423n4/2022-05-opensea-seaport/blob/9d7ce4d08bf3c3010304a0476a785c70c0e90ae7/contracts/lib/TokenTransferrer.sol#L35-L39) can benefit from using readable constants instead of hex/numeric literals

*There are 112 instances of this issue:*
```solidity
File: src/ArtGobblers.sol

/// @audit 69.42e18
304:              69.42e18, // Target price.

/// @audit 0.31e18
305:              0.31e18, // Price decay percent.

/// @audit 0.0023e18
308:              0.0023e18 // Time scale.

/// @audit 9
632:                  uint256 newCurrentIdMultiple = 9; // For beyond 7963.

/// @audit 5
848:              if (newNumMintedForReserves > (numMintedFromGoo + newNumMintedForReserves) / 5) revert ReserveImbalance();

```
https://github.com/code-423n4/2022-09-artgobblers/blob/d2087c5a8a6a4f1b9784520e7fe75afa3a9cbdbe/src/ArtGobblers.sol#L304

```solidity
File: lib/solmate/src/utils/FixedPointMathLib.sol

/// @audit 0x10000000000000000000000000000000000
177:              if iszero(lt(y, 0x10000000000000000000000000000000000)) {

/// @audit 0x1000000000000000000
181:              if iszero(lt(y, 0x1000000000000000000)) {

/// @audit 0x10000000000
185:              if iszero(lt(y, 0x10000000000)) {

/// @audit 0x1000000
189:              if iszero(lt(y, 0x1000000)) {

```
https://github.com/transmissions11/solmate/blob/26572802743101f160f2d07556edfc162896115e/src/utils/FixedPointMathLib.sol#L177

```solidity
File: lib/solmate/src/tokens/ERC721.sol

/// @audit 0x01ffc9a7
148:              interfaceId == 0x01ffc9a7 || // ERC165 Interface ID for ERC165

/// @audit 0x80ac58cd
149:              interfaceId == 0x80ac58cd || // ERC165 Interface ID for ERC721

/// @audit 0x5b5e139f
150:              interfaceId == 0x5b5e139f; // ERC165 Interface ID for ERC721Metadata

```
https://github.com/transmissions11/solmate/blob/26572802743101f160f2d07556edfc162896115e/src/tokens/ERC721.sol#L148

```solidity
File: src/utils/token/GobblersERC1155B.sol

/// @audit 0x01ffc9a7
126:              interfaceId == 0x01ffc9a7 || // ERC165 Interface ID for ERC165

/// @audit 0xd9b67a26
127:              interfaceId == 0xd9b67a26 || // ERC165 Interface ID for ERC1155

/// @audit 0x0e89341c
128:              interfaceId == 0x0e89341c; // ERC165 Interface ID for ERC1155MetadataURI

```
https://github.com/code-423n4/2022-09-artgobblers/blob/d2087c5a8a6a4f1b9784520e7fe75afa3a9cbdbe/src/utils/token/GobblersERC1155B.sol#L126

```solidity
File: lib/solmate/src/utils/SignedWadMath.sol

/// @audit 42139678854452767551
86:           if (x <= -42139678854452767551) return 0;

/// @audit 135305999368893231589
90:           if (x >= 135305999368893231589) revert("EXP_OVERFLOW");

/// @audit 78
/// @audit 5
/// @audit 18
95:           x = (x << 78) / 5**18;

/// @audit 96
/// @audit 54916777467707473351141471128
/// @audit 95
/// @audit 96
100:          int256 k = ((x << 96) / 54916777467707473351141471128 + 2**95) >> 96;

/// @audit 54916777467707473351141471128
101:          x = x - k * 54916777467707473351141471128;

/// @audit 1346386616545796478920950773328
107:          int256 y = x + 1346386616545796478920950773328;

/// @audit 96
/// @audit 57155421227552351082224309758442
108:          y = ((y * x) >> 96) + 57155421227552351082224309758442;

/// @audit 94201549194550492254356042504812
109:          int256 p = y + x - 94201549194550492254356042504812;

/// @audit 96
/// @audit 28719021644029726153956944680412240
110:          p = ((p * y) >> 96) + 28719021644029726153956944680412240;

/// @audit 4385272521454847904659076985693276
/// @audit 96
111:          p = p * x + (4385272521454847904659076985693276 << 96);

/// @audit 2855989394907223263936484059900
114:          int256 q = x - 2855989394907223263936484059900;

/// @audit 96
/// @audit 50020603652535783019961831881945
115:          q = ((q * x) >> 96) + 50020603652535783019961831881945;

/// @audit 96
/// @audit 533845033583426703283633433725380
116:          q = ((q * x) >> 96) - 533845033583426703283633433725380;

/// @audit 96
/// @audit 3604857256930695427073651918091429
117:          q = ((q * x) >> 96) + 3604857256930695427073651918091429;

/// @audit 96
/// @audit 14423608567350463180887372962807573
118:          q = ((q * x) >> 96) - 14423608567350463180887372962807573;

/// @audit 96
/// @audit 26449188498355588339934803723976023
119:          q = ((q * x) >> 96) + 26449188498355588339934803723976023;

/// @audit 3822833074963236453042738258902158003155416615667
/// @audit 195
136:          r = int256((uint256(r) * 3822833074963236453042738258902158003155416615667) >> uint256(195 - k));

/// @audit 0xffffffffffffffffffffffffffffffff
150:              r := shl(7, lt(0xffffffffffffffffffffffffffffffff, x))

/// @audit 0xffffffffffffffff
151:              r := or(r, shl(6, lt(0xffffffffffffffff, shr(r, x))))

/// @audit 0xffffffff
152:              r := or(r, shl(5, lt(0xffffffff, shr(r, x))))

/// @audit 0xffff
153:              r := or(r, shl(4, lt(0xffff, shr(r, x))))

/// @audit 0xff
154:              r := or(r, shl(3, lt(0xff, shr(r, x))))

/// @audit 0xf
155:              r := or(r, shl(2, lt(0xf, shr(r, x))))

/// @audit 0x3
156:              r := or(r, shl(1, lt(0x3, shr(r, x))))

/// @audit 0x1
157:              r := or(r, lt(0x1, shr(r, x)))

/// @audit 96
162:          int256 k = r - 96;

/// @audit 159
163:          x <<= uint256(159 - k);

/// @audit 159
164:          x = int256(uint256(x) >> 159);

/// @audit 3273285459638523848632254066296
168:          int256 p = x + 3273285459638523848632254066296;

/// @audit 96
/// @audit 24828157081833163892658089445524
169:          p = ((p * x) >> 96) + 24828157081833163892658089445524;

/// @audit 96
/// @audit 43456485725739037958740375743393
170:          p = ((p * x) >> 96) + 43456485725739037958740375743393;

/// @audit 96
/// @audit 11111509109440967052023855526967
171:          p = ((p * x) >> 96) - 11111509109440967052023855526967;

/// @audit 96
/// @audit 45023709667254063763336534515857
172:          p = ((p * x) >> 96) - 45023709667254063763336534515857;

/// @audit 96
/// @audit 14706773417378608786704636184526
173:          p = ((p * x) >> 96) - 14706773417378608786704636184526;

/// @audit 795164235651350426258249787498
/// @audit 96
174:          p = p * x - (795164235651350426258249787498 << 96);

/// @audit 5573035233440673466300451813936
178:          int256 q = x + 5573035233440673466300451813936;

/// @audit 96
/// @audit 71694874799317883764090561454958
179:          q = ((q * x) >> 96) + 71694874799317883764090561454958;

/// @audit 96
/// @audit 283447036172924575727196451306956
180:          q = ((q * x) >> 96) + 283447036172924575727196451306956;

/// @audit 96
/// @audit 401686690394027663651624208769553
181:          q = ((q * x) >> 96) + 401686690394027663651624208769553;

/// @audit 96
/// @audit 204048457590392012362485061816622
182:          q = ((q * x) >> 96) + 204048457590392012362485061816622;

/// @audit 96
/// @audit 31853899698501571402653359427138
183:          q = ((q * x) >> 96) + 31853899698501571402653359427138;

/// @audit 96
/// @audit 909429971244387300277376558375
184:          q = ((q * x) >> 96) + 909429971244387300277376558375;

/// @audit 1677202110996718588342820967067443963516166
201:          r *= 1677202110996718588342820967067443963516166;

/// @audit 16597577552685614221487285958193947469193820559219878177908093499208371
203:          r += 16597577552685614221487285958193947469193820559219878177908093499208371 * k;

/// @audit 600920179829731861736702779321621459595472258049074101567377883020018308
205:          r += 600920179829731861736702779321621459595472258049074101567377883020018308;

/// @audit 174
207:          r >>= 174;

```
https://github.com/transmissions11/solmate/blob/26572802743101f160f2d07556edfc162896115e/src/utils/SignedWadMath.sol#L86

```solidity
File: src/utils/token/GobblersERC721.sol

/// @audit 0x01ffc9a7
151:              interfaceId == 0x01ffc9a7 || // ERC165 Interface ID for ERC165

/// @audit 0x80ac58cd
152:              interfaceId == 0x80ac58cd || // ERC165 Interface ID for ERC721

/// @audit 0x5b5e139f
153:              interfaceId == 0x5b5e139f; // ERC165 Interface ID for ERC721Metadata

```
https://github.com/code-423n4/2022-09-artgobblers/blob/d2087c5a8a6a4f1b9784520e7fe75afa3a9cbdbe/src/utils/token/GobblersERC721.sol#L151

```solidity
File: src/utils/token/PagesERC721.sol

/// @audit 0x01ffc9a7
164:              interfaceId == 0x01ffc9a7 || // ERC165 Interface ID for ERC165

/// @audit 0x80ac58cd
165:              interfaceId == 0x80ac58cd || // ERC165 Interface ID for ERC721

/// @audit 0x5b5e139f
166:              interfaceId == 0x5b5e139f; // ERC165 Interface ID for ERC721Metadata

```
https://github.com/code-423n4/2022-09-artgobblers/blob/d2087c5a8a6a4f1b9784520e7fe75afa3a9cbdbe/src/utils/token/PagesERC721.sol#L164

```solidity
File: script/deploy/DeployBase.s.sol

/// @audit 4
66:           address gobblerAddress = LibRLP.computeAddress(tx.origin, vm.getNonce(tx.origin) + 4);

/// @audit 5
67:           address pageAddress = LibRLP.computeAddress(tx.origin, vm.getNonce(tx.origin) + 5);

```
https://github.com/code-423n4/2022-09-artgobblers/blob/d2087c5a8a6a4f1b9784520e7fe75afa3a9cbdbe/script/deploy/DeployBase.s.sol#L66

```solidity
File: src/Pages.sol

/// @audit 4.2069e18
168:              4.2069e18, // Target price.

/// @audit 0.31e18
169:              0.31e18, // Price decay percent.

/// @audit 9000e18
170:              9000e18, // Logistic asymptote.

/// @audit 0.014e18
171:              0.014e18, // Logistic time scale.

/// @audit 9e18
174:              9e18 // Pages to target per day.

```
https://github.com/code-423n4/2022-09-artgobblers/blob/d2087c5a8a6a4f1b9784520e7fe75afa3a9cbdbe/src/Pages.sol#L168

```solidity
File: script/deploy/DeployRinkeby.s.sol

/// @audit 0xb3dCcb4Cf7a26f6cf6B120Cf5A73875B7BBc655B
26:               address(0xb3dCcb4Cf7a26f6cf6B120Cf5A73875B7BBc655B),

/// @audit 0x01BE23585060835E02B77ef475b0Cc51aA1e0709
28:               address(0x01BE23585060835E02B77ef475b0Cc51aA1e0709),

/// @audit 0x2ed0feb3e7fd2022120aa84fab1945545a9f2ffc9076fd6156fa96eaff4c1311
30:               0x2ed0feb3e7fd2022120aa84fab1945545a9f2ffc9076fd6156fa96eaff4c1311,

/// @audit 0.1e18
32:               0.1e18,

```
https://github.com/code-423n4/2022-09-artgobblers/blob/d2087c5a8a6a4f1b9784520e7fe75afa3a9cbdbe/script/deploy/DeployRinkeby.s.sol#L26

```solidity
File: src/Goo.sol

/// @audit 18
58:   contract Goo is ERC20("Goo", "GOO", 18) {

```
https://github.com/code-423n4/2022-09-artgobblers/blob/d2087c5a8a6a4f1b9784520e7fe75afa3a9cbdbe/src/Goo.sol#L58

```solidity
File: lib/VRGDAs/src/LogisticVRGDA.sol

/// @audit 1e18
44:           logisticLimit = _maxSellable + 1e18;

/// @audit 2e18
47:           logisticLimitDoubled = logisticLimit * 2e18;

/// @audit 1e18
62:               return -unsafeWadDiv(wadLn(unsafeDiv(logisticLimitDoubled, sold + logisticLimit) - 1e18), timeScale);

```
https://github.com/transmissions11/VRGDAs/blob/f4dec0611641e344339b2c78e5f733bba3b532e0/src/LogisticVRGDA.sol#L44

```solidity
File: lib/solmate/src/utils/LibString.sol

/// @audit 0x40
13:               let newFreeMemoryPointer := add(mload(0x40), 160)

/// @audit 0x40
16:               mstore(0x40, newFreeMemoryPointer)

```
https://github.com/transmissions11/solmate/blob/26572802743101f160f2d07556edfc162896115e/src/utils/LibString.sol#L13

```solidity
File: lib/VRGDAs/src/VRGDA.sol

/// @audit 1e18
29:           decayConstant = wadLn(1e18 - _priceDecayPercent);

```
https://github.com/transmissions11/VRGDAs/blob/f4dec0611641e344339b2c78e5f733bba3b532e0/src/VRGDA.sol#L29

```solidity
File: lib/goo-issuance/src/LibGOO.sol

/// @audit 1e18
38:                   (emissionMultiple * lastBalanceWad * 1e18).sqrt()

```
https://github.com/transmissions11/goo-issuance/blob/648e65e66e43ff5c19681427e300ece9c0df1437/src/LibGOO.sol#L38

### [N&#x2011;06]  Use a more recent version of solidity
Use a solidity version of at least 0.8.13 to get the ability to use `using for` with a list of free functions

*There are 2 instances of this issue:*
```solidity
File: src/Pages.sol

2:    pragma solidity >=0.8.0;

```
https://github.com/code-423n4/2022-09-artgobblers/blob/d2087c5a8a6a4f1b9784520e7fe75afa3a9cbdbe/src/Pages.sol#L2

```solidity
File: lib/goo-issuance/src/LibGOO.sol

2:    pragma solidity >=0.8.0;

```
https://github.com/transmissions11/goo-issuance/blob/648e65e66e43ff5c19681427e300ece9c0df1437/src/LibGOO.sol#L2

### [N&#x2011;07]  Use a more recent version of solidity
Use a solidity version of at least 0.8.4 to get `bytes.concat()` instead of `abi.encodePacked(<bytes>,<bytes>)`
Use a solidity version of at least 0.8.12 to get `string.concat()` instead of `abi.encodePacked(<str>,<str>)`

*There are 2 instances of this issue:*
```solidity
File: src/ArtGobblers.sol

2:    pragma solidity >=0.8.0;

```
https://github.com/code-423n4/2022-09-artgobblers/blob/d2087c5a8a6a4f1b9784520e7fe75afa3a9cbdbe/src/ArtGobblers.sol#L2

```solidity
File: script/deploy/DeployRinkeby.s.sol

2:    pragma solidity >=0.8.0;

```
https://github.com/code-423n4/2022-09-artgobblers/blob/d2087c5a8a6a4f1b9784520e7fe75afa3a9cbdbe/script/deploy/DeployRinkeby.s.sol#L2

### [N&#x2011;08]  Constant redefined elsewhere
Consider defining in only one contract so that values cannot become out of sync when only one location is updated. A [cheap way](https://medium.com/coinmonks/gas-cost-of-solidity-library-functions-dbe0cedd4678) to store constants in a single location is to create an `internal constant` in a `library`. If the variable is a local cache of another contract's value, consider making the cache variable internal or private, which will require external users to query the contract with the source of truth, so that callers don't get out of sync.

*There are 8 instances of this issue:*
```solidity
File: src/Pages.sol

/// @audit seen in src/ArtGobblers.sol 
86:       Goo public immutable goo;

/// @audit seen in src/ArtGobblers.sol 
89:       address public immutable community;

/// @audit seen in src/ArtGobblers.sol 
103:      uint256 public immutable mintStart;

```
https://github.com/code-423n4/2022-09-artgobblers/blob/d2087c5a8a6a4f1b9784520e7fe75afa3a9cbdbe/src/Pages.sol#L86

```solidity
File: src/utils/rand/ChainlinkV1RandProvider.sol

/// @audit seen in src/utils/token/PagesERC721.sol 
20:       ArtGobblers public immutable artGobblers;

```
https://github.com/code-423n4/2022-09-artgobblers/blob/d2087c5a8a6a4f1b9784520e7fe75afa3a9cbdbe/src/utils/rand/ChainlinkV1RandProvider.sol#L20

```solidity
File: script/deploy/DeployRinkeby.s.sol

/// @audit seen in src/Pages.sol 
11:       uint256 public immutable mintStart = 1656369768;

```
https://github.com/code-423n4/2022-09-artgobblers/blob/d2087c5a8a6a4f1b9784520e7fe75afa3a9cbdbe/script/deploy/DeployRinkeby.s.sol#L11

```solidity
File: src/Goo.sol

/// @audit seen in src/utils/rand/ChainlinkV1RandProvider.sol 
64:       address public immutable artGobblers;

/// @audit seen in src/ArtGobblers.sol 
67:       address public immutable pages;

```
https://github.com/code-423n4/2022-09-artgobblers/blob/d2087c5a8a6a4f1b9784520e7fe75afa3a9cbdbe/src/Goo.sol#L64

```solidity
File: src/utils/GobblerReserve.sol

/// @audit seen in src/Goo.sol 
18:       ArtGobblers public immutable artGobblers;

```
https://github.com/code-423n4/2022-09-artgobblers/blob/d2087c5a8a6a4f1b9784520e7fe75afa3a9cbdbe/src/utils/GobblerReserve.sol#L18

### [N&#x2011;09]  Variable names that consist of all capital letters should be reserved for `constant`/`immutable` variables
If the variable needs to be different based on which class it comes from, a `view`/`pure` _function_ should be used instead (e.g. like [this](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/76eee35971c2541585e05cbf258510dda7b2fbc6/contracts/token/ERC20/extensions/draft-IERC20Permit.sol#L59)).

*There are 3 instances of this issue:*
```solidity
File: src/ArtGobblers.sol

136:      string public UNREVEALED_URI;

139:      string public BASE_URI;

```
https://github.com/code-423n4/2022-09-artgobblers/blob/d2087c5a8a6a4f1b9784520e7fe75afa3a9cbdbe/src/ArtGobblers.sol#L136

```solidity
File: src/Pages.sol

96:       string public BASE_URI;

```
https://github.com/code-423n4/2022-09-artgobblers/blob/d2087c5a8a6a4f1b9784520e7fe75afa3a9cbdbe/src/Pages.sol#L96

### [N&#x2011;10]  File is missing NatSpec

*There are 2 instances of this issue:*
```solidity
File: script/deploy/DeployBase.s.sol


```
https://github.com/code-423n4/2022-09-artgobblers/blob/d2087c5a8a6a4f1b9784520e7fe75afa3a9cbdbe/script/deploy/DeployBase.s.sol

```solidity
File: script/deploy/DeployRinkeby.s.sol


```
https://github.com/code-423n4/2022-09-artgobblers/blob/d2087c5a8a6a4f1b9784520e7fe75afa3a9cbdbe/script/deploy/DeployRinkeby.s.sol

### [N&#x2011;11]  NatSpec is incomplete

*There are 10 instances of this issue:*
```solidity
File: src/ArtGobblers.sol

/// @audit Missing: '@param _pages'
278       /// @notice Sets VRGDA parameters, mint config, relevant addresses, and URIs.
279       /// @param _merkleRoot Merkle root of mint mintlist.
280       /// @param _mintStart Timestamp for the start of the VRGDA mint.
281       /// @param _goo Address of the Goo contract.
282       /// @param _team Address of the team reserve.
283       /// @param _community Address of the community reserve.
284       /// @param _randProvider Address of the randomness provider.
285       /// @param _baseUri Base URI for revealed gobblers.
286       /// @param _unrevealedUri URI for unrevealed gobblers.
287       constructor(
288           // Mint config:
289           bytes32 _merkleRoot,
290           uint256 _mintStart,
291           // Addresses:
292           Goo _goo,
293           Pages _pages,
294           address _team,
295           address _community,
296           RandProvider _randProvider,
297           // URIs:
298           string memory _baseUri,
299           string memory _unrevealedUri
300       )
301           GobblersERC721("Art Gobblers", "GOBBLER")
302           Owned(msg.sender)
303           LogisticVRGDA(
304               69.42e18, // Target price.
305               0.31e18, // Price decay percent.
306               // Max gobblers mintable via VRGDA.
307               toWadUnsafe(MAX_MINTABLE),
308               0.0023e18 // Time scale.
309:          )

/// @audit Missing: '@param bytes32'
544       /// @notice Callback from rand provider. Sets randomSeed. Can only be called by the rand provider.
545       /// @param randomness The 256 bits of verifiable randomness provided by the rand provider.
546:      function acceptRandomSeed(bytes32, uint256 randomness) external {

/// @audit Missing: '@return'
691       /// @notice Returns a token's URI if it has been minted.
692       /// @param gobblerId The id of the token to get the URI for.
693:      function tokenURI(uint256 gobblerId) public view virtual override returns (string memory) {

/// @audit Missing: '@return'
755       /// @notice Calculate a user's virtual goo balance.
756       /// @param user The user to query balance for.
757:      function gooBalance(address user) public view returns (uint256) {

/// @audit Missing: '@return'
837       /// @dev Gobblers minted to reserves cannot comprise more than 20% of the sum of
838       /// the supply of goo minted gobblers and the supply of gobblers minted to reserves.
839:      function mintReservedGobblers(uint256 numGobblersEach) external returns (uint256 lastMintedGobblerId) {

/// @audit Missing: '@return'
864       /// @notice Convenience function to get emissionMultiple for a gobbler.
865       /// @param gobblerId The gobbler to get emissionMultiple for.
866:      function getGobblerEmissionMultiple(uint256 gobblerId) external view returns (uint256) {

/// @audit Missing: '@return'
870       /// @notice Convenience function to get emissionMultiple for a user.
871       /// @param user The user to get emissionMultiple for.
872:      function getUserEmissionMultiple(address user) external view returns (uint256) {

```
https://github.com/code-423n4/2022-09-artgobblers/blob/d2087c5a8a6a4f1b9784520e7fe75afa3a9cbdbe/src/ArtGobblers.sol#L278-L309

```solidity
File: src/Pages.sol

/// @audit Missing: '@return'
237       /// @dev Pages minted to the reserve cannot comprise more than 10% of the sum of the
238       /// supply of goo minted pages and the supply of pages minted to the community reserve.
239:      function mintCommunityPages(uint256 numPages) external returns (uint256 lastMintedPageId) {

/// @audit Missing: '@return'
263       /// @notice Returns a page's URI if it has been minted.
264       /// @param pageId The id of the page to get the URI for.
265:      function tokenURI(uint256 pageId) public view virtual override returns (string memory) {

```
https://github.com/code-423n4/2022-09-artgobblers/blob/d2087c5a8a6a4f1b9784520e7fe75afa3a9cbdbe/src/Pages.sol#L237-L239

```solidity
File: lib/goo-issuance/src/LibGOO.sol

/// @audit Missing: '@return'
15        /// @param lastBalanceWad The last checkpointed balance to apply the emission multiple over time to, scaled by 1e18.
16        /// @param timeElapsedWad The time elapsed since the last checkpoint, scaled by 1e18.
17        function computeGOOBalance(
18            uint256 emissionMultiple,
19            uint256 lastBalanceWad,
20            uint256 timeElapsedWad
21:       ) public pure returns (uint256) {

```
https://github.com/transmissions11/goo-issuance/blob/648e65e66e43ff5c19681427e300ece9c0df1437/src/LibGOO.sol#L15-L21

### [N&#x2011;12]  Event is missing `indexed` fields
Index event fields make the field more quickly accessible to off-chain tools that parse events. However, note that each index field costs extra gas during emission, so it's not necessarily best to index the maximum allowed per event (three fields). Each `event` should use three `indexed` fields if there are three or more fields, and gas usage is not particularly of concern for the events in question. If there are fewer than three fields, all of the fields should be indexed.

*There are 16 instances of this issue:*
```solidity
File: src/ArtGobblers.sol

229:      event GooBalanceUpdated(address indexed user, uint256 newGooBalance);

232:      event GobblerPurchased(address indexed user, uint256 indexed gobblerId, uint256 price);

233:      event LegendaryGobblerMinted(address indexed user, uint256 indexed gobblerId, uint256[] burnedGobblerIds);

234:      event ReservedGobblersMinted(address indexed user, uint256 lastMintedGobblerId, uint256 numGobblersEach);

236:      event RandomnessFulfilled(uint256 randomness);

237:      event RandomnessRequested(address indexed user, uint256 toBeRevealed);

240:      event GobblersRevealed(address indexed user, uint256 numGobblers, uint256 lastRevealedId);

```
https://github.com/code-423n4/2022-09-artgobblers/blob/d2087c5a8a6a4f1b9784520e7fe75afa3a9cbdbe/src/ArtGobblers.sol#L229

```solidity
File: lib/solmate/src/tokens/ERC721.sol

15:       event ApprovalForAll(address indexed owner, address indexed operator, bool approved);

```
https://github.com/transmissions11/solmate/blob/26572802743101f160f2d07556edfc162896115e/src/tokens/ERC721.sol#L15

```solidity
File: src/utils/token/GobblersERC1155B.sol

29:       event ApprovalForAll(address indexed owner, address indexed operator, bool approved);

31:       event URI(string value, uint256 indexed id);

```
https://github.com/code-423n4/2022-09-artgobblers/blob/d2087c5a8a6a4f1b9784520e7fe75afa3a9cbdbe/src/utils/token/GobblersERC1155B.sol#L29

```solidity
File: src/utils/token/GobblersERC721.sol

17:       event ApprovalForAll(address indexed owner, address indexed operator, bool approved);

```
https://github.com/code-423n4/2022-09-artgobblers/blob/d2087c5a8a6a4f1b9784520e7fe75afa3a9cbdbe/src/utils/token/GobblersERC721.sol#L17

```solidity
File: src/utils/token/PagesERC721.sol

18:       event ApprovalForAll(address indexed owner, address indexed operator, bool approved);

```
https://github.com/code-423n4/2022-09-artgobblers/blob/d2087c5a8a6a4f1b9784520e7fe75afa3a9cbdbe/src/utils/token/PagesERC721.sol#L18

```solidity
File: src/Pages.sol

134:      event PagePurchased(address indexed user, uint256 indexed pageId, uint256 price);

136:      event CommunityPagesMinted(address indexed user, uint256 lastMintedPageId, uint256 numPages);

```
https://github.com/code-423n4/2022-09-artgobblers/blob/d2087c5a8a6a4f1b9784520e7fe75afa3a9cbdbe/src/Pages.sol#L134

```solidity
File: src/utils/rand/RandProvider.sol

13:       event RandomBytesRequested(bytes32 requestId);

14:       event RandomBytesReturned(bytes32 requestId, uint256 randomness);

```
https://github.com/code-423n4/2022-09-artgobblers/blob/d2087c5a8a6a4f1b9784520e7fe75afa3a9cbdbe/src/utils/rand/RandProvider.sol#L13

### [N&#x2011;13]  Not using the named return variables anywhere in the function is confusing
Consider changing the variable to be an unnamed one

*There are 3 instances of this issue:*
```solidity
File: src/utils/token/GobblersERC1155B.sol

/// @audit owner
55:       function ownerOf(uint256 id) public view virtual returns (address owner) {

```
https://github.com/code-423n4/2022-09-artgobblers/blob/d2087c5a8a6a4f1b9784520e7fe75afa3a9cbdbe/src/utils/token/GobblersERC1155B.sol#L55

```solidity
File: src/utils/token/PagesERC721.sol

/// @audit isApproved
72:       function isApprovedForAll(address owner, address operator) public view returns (bool isApproved) {

```
https://github.com/code-423n4/2022-09-artgobblers/blob/d2087c5a8a6a4f1b9784520e7fe75afa3a9cbdbe/src/utils/token/PagesERC721.sol#L72

```solidity
File: src/utils/rand/ChainlinkV1RandProvider.sol

/// @audit requestId
62:       function requestRandomBytes() external returns (bytes32 requestId) {

```
https://github.com/code-423n4/2022-09-artgobblers/blob/d2087c5a8a6a4f1b9784520e7fe75afa3a9cbdbe/src/utils/rand/ChainlinkV1RandProvider.sol#L62

### [N&#x2011;14]  Duplicated `require()`/`revert()` checks should be refactored to a modifier or function
The compiler will inline the function, which will avoid `JUMP` instructions usually associated with functions

*There are 2 instances of this issue:*
```solidity
File: src/ArtGobblers.sol

705:          if (gobblerId < FIRST_LEGENDARY_GOBBLER_ID) revert("NOT_MINTED");

```
https://github.com/code-423n4/2022-09-artgobblers/blob/d2087c5a8a6a4f1b9784520e7fe75afa3a9cbdbe/src/ArtGobblers.sol#L705

```solidity
File: lib/solmate/src/tokens/ERC721.sol

158:          require(to != address(0), "INVALID_RECIPIENT");

```
https://github.com/transmissions11/solmate/blob/26572802743101f160f2d07556edfc162896115e/src/tokens/ERC721.sol#L158



