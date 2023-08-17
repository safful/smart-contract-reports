## Tags

- bug
- QA (Quality Assurance)
- grade-a
- selected for report
- Q-10

# [QA Report](https://github.com/code-423n4/2022-10-inverse-findings/issues/126) 

- [Low](#low)
    - [**1. Allows malleable SECP256K1 signatures**](#1-allows-malleable-secp256k1-signatures)
    - [**2. Lack of checks address0**](#2-lack-of-checks-address0)
    - [**3. Avoid using tx.origin**](#3-avoid-using-txorigin)
    - [**4. Mixing and Outdated compiler**](#4-mixing-and-outdated-compiler)
    - [**5. Lack of ACK during owner change**](#5-lack-of-ack-during-owner-change)
    - [**6. Market pause is not checked during contraction**](#6-market-pause-is-not-checked-during-contraction)
    - [**7. Lack of no reentrant modifier**](#7-lack-of-no-reentrant-modifier)
    - [**8. Lack of checks the integer ranges**](#8-lack-of-checks-the-integer-ranges)
    - [**9. Lack of checks supportsInterface**](#9-lack-of-checks-supportsinterface)
    - [**10. Lack of event emit**](#10-lack-of-event-emit)
    - [**11. Oracle not compatible with tokens of 19 or more decimals**](#11-oracle-not-compatible-with-tokens-of-19-or-more-decimals)
    - [**12. Wrong visibility**](#12-wrong-visibility)
- [Non critical](#non-critical)
    - [**13. Bad nomenclature**](#13-bad-nomenclature)
    - [**14. Open TODO**](#14-open-todo)
    - [**15. Avoid duplicate code**](#15-avoid-duplicate-code)
    - [**16. Avoid hardcoded values**](#16-avoid-hardcoded-values)

# Low

## **1. Allows malleable `SECP256K1` signatures**

Here, the `ecrecover()` method doesn't check the `s` range.

Homestead ([EIP-2](https://eips.ethereum.org/EIPS/eip-2)) added this limitation, however the precompile remained unaltered. The majority of libraries, including OpenZeppelin, do this check.

Since an order can only be confirmed once and its hash is saved, there doesn't seem to be a serious danger in existing use cases.

**Reference:**

- https://github.com/OpenZeppelin/openzeppelin-contracts/blob/7201e6707f6631d9499a569f492870ebdd4133cf/contracts/utils/cryptography/ECDSA.sol#L138-L149

**Affected source code:**

- [DBR.sol:226-248](https://github.com/code-423n4/2022-10-inverse/blob/3e81f0f5908ea99b36e6ab72f13488bbfe622183/src/DBR.sol#L226-L248)
- [Market.sol:425-447](https://github.com/code-423n4/2022-10-inverse/blob/3e81f0f5908ea99b36e6ab72f13488bbfe622183/src/Market.sol#L425-L447)
- [Market.sol:489-511](https://github.com/code-423n4/2022-10-inverse/blob/3e81f0f5908ea99b36e6ab72f13488bbfe622183/src/Market.sol#L489-L511)

## **2. Lack of checks `address(0)`**

The following methods have a lack of checks if the received argument is an address, it's good practice in order to reduce human error to check that the address specified in the constructor or initialize is different than `address(0)`.

**Affected source code:**

- [BorrowController.sol:14](https://github.com/code-423n4/2022-10-inverse/blob/3e81f0f5908ea99b36e6ab72f13488bbfe622183/src/BorrowController.sol#L14)
- [BorrowController.sol:26](https://github.com/code-423n4/2022-10-inverse/blob/3e81f0f5908ea99b36e6ab72f13488bbfe622183/src/BorrowController.sol#L26)
- [SimpleERC20Escrow.sol:28](https://github.com/code-423n4/2022-10-inverse/blob/3e81f0f5908ea99b36e6ab72f13488bbfe622183/src/escrows/SimpleERC20Escrow.sol#L28)
- [GovTokenEscrow.sol:33-34](https://github.com/code-423n4/2022-10-inverse/blob/3e81f0f5908ea99b36e6ab72f13488bbfe622183/src/escrows/GovTokenEscrow.sol#L33-L34)
- [INVEscrow.sol:35](https://github.com/code-423n4/2022-10-inverse/blob/3e81f0f5908ea99b36e6ab72f13488bbfe622183/src/escrows/INVEscrow.sol#L35)
- [INVEscrow.sol:47-48](https://github.com/code-423n4/2022-10-inverse/blob/3e81f0f5908ea99b36e6ab72f13488bbfe622183/src/escrows/INVEscrow.sol#L47-L48)
- [Fed.sol:37-40](https://github.com/code-423n4/2022-10-inverse/blob/3e81f0f5908ea99b36e6ab72f13488bbfe622183/src/Fed.sol#L37-L40)
- [Fed.sol:50](https://github.com/code-423n4/2022-10-inverse/blob/3e81f0f5908ea99b36e6ab72f13488bbfe622183/src/Fed.sol#L50)
- [Fed.sol:68](https://github.com/code-423n4/2022-10-inverse/blob/3e81f0f5908ea99b36e6ab72f13488bbfe622183/src/Fed.sol#L68)
- [Oracle.sol:32](https://github.com/code-423n4/2022-10-inverse/blob/3e81f0f5908ea99b36e6ab72f13488bbfe622183/src/Oracle.sol#L32)
- [Oracle.sol:44](https://github.com/code-423n4/2022-10-inverse/blob/3e81f0f5908ea99b36e6ab72f13488bbfe622183/src/Oracle.sol#L44)
- [DBR.sol:39](https://github.com/code-423n4/2022-10-inverse/blob/3e81f0f5908ea99b36e6ab72f13488bbfe622183/src/DBR.sol#L39)
- [DBR.sol:54](https://github.com/code-423n4/2022-10-inverse/blob/3e81f0f5908ea99b36e6ab72f13488bbfe622183/src/DBR.sol#L54)
- [Market.sol:77-83](https://github.com/code-423n4/2022-10-inverse/blob/3e81f0f5908ea99b36e6ab72f13488bbfe622183/src/Market.sol#L77-L83)
- [Market.sol:130](https://github.com/code-423n4/2022-10-inverse/blob/3e81f0f5908ea99b36e6ab72f13488bbfe622183/src/Market.sol#L130)
- [Market.sol:136](https://github.com/code-423n4/2022-10-inverse/blob/3e81f0f5908ea99b36e6ab72f13488bbfe622183/src/Market.sol#L136)
- [Market.sol:142](https://github.com/code-423n4/2022-10-inverse/blob/3e81f0f5908ea99b36e6ab72f13488bbfe622183/src/Market.sol#L142)

## **3. Avoid using `tx.origin`**

`tx.origin` is a global variable in Solidity that returns the address of the account that sent the transaction.

Using the variable could make a contract vulnerable if an authorized account calls a malicious contract. You can impersonate a user using a third party contract.

This can make it easier to create a vault on behalf of another user with an external administrator (by receiving it as an argument).

**Affected source code:**

- [BorrowController.sol:47](https://github.com/code-423n4/2022-10-inverse/blob/3e81f0f5908ea99b36e6ab72f13488bbfe622183/src/BorrowController.sol#L47)

## **4. Mixing and Outdated compiler**

The pragma version used are:

```
pragma solidity ^0.8.13;
```

*Note that mixing pragma is not recommended. Because different compiler versions have different meanings and behaviors, it also significantly raises maintenance costs. As a result, depending on the compiler version selected for any given file, deployed contracts may have security issues.*

The minimum required version must be [0.8.17](https://github.com/ethereum/solidity/releases/tag/v0.8.17); otherwise, contracts will be affected by the following **important bug fixes**:

[0.8.14](https://blog.soliditylang.org/2022/05/18/solidity-0.8.14-release-announcement/):

- ABI Encoder: When ABI-encoding values from calldata that contain nested arrays, correctly validate the nested array length against `calldatasize()` in all cases.
- Override Checker: Allow changing data location for parameters only when overriding external functions.

[0.8.15](https://blog.soliditylang.org/2022/06/15/solidity-0.8.15-release-announcement/)

- Code Generation: Avoid writing dirty bytes to storage when copying `bytes` arrays.
- Yul Optimizer: Keep all memory side-effects of inline assembly blocks.

[0.8.16](https://blog.soliditylang.org/2022/08/08/solidity-0.8.16-release-announcement/)

- Code Generation: Fix data corruption that affected ABI-encoding of calldata values represented by tuples: structs at any nesting level; argument lists of external functions, events and errors; return value lists of external functions. The 32 leading bytes of the first dynamically-encoded value in the tuple would get zeroed when the last component contained a statically-encoded array.

[0.8.17](https://blog.soliditylang.org/2022/09/08/solidity-0.8.17-release-announcement/)

- Yul Optimizer: Prevent the incorrect removal of storage writes before calls to Yul functions that conditionally terminate the external EVM call.

Apart from these, there are several minor bug fixes and improvements.

## **5. Lack of ACK during owner change**

It's possible to lose the ownership under specific circumstances.

Because of human error it's possible to set a new invalid owner. When you want to change the owner's address it's better to propose a new owner, and then accept this ownership with the new wallet.

**Affected source code:**

- [Fed.sol:50](https://github.com/code-423n4/2022-10-inverse/blob/3e81f0f5908ea99b36e6ab72f13488bbfe622183/src/Fed.sol#L50)
- [Market.sol:130](https://github.com/code-423n4/2022-10-inverse/blob/3e81f0f5908ea99b36e6ab72f13488bbfe622183/src/Market.sol#L130)

## **6. Market pause is not checked during `contraction`**

In the `Fed` contract, during the `expansion` method is checked that the market is not paused, this requirement is not done during the `contraction`.

```diff
    function contraction(IMarket market, uint amount) public {
        require(msg.sender == chair, "ONLY CHAIR");
        require(dbr.markets(address(market)), "UNSUPPORTED MARKET");
+       require(!market.borrowPaused(), "CANNOT EXPAND PAUSED MARKETS");
        uint supply = supplies[market];
        require(amount <= supply, "AMOUNT TOO BIG"); // can't burn profits
        market.recall(amount);
        dola.burn(amount);
        supplies[market] -= amount;
        globalSupply -= amount;
        emit Contraction(market, amount);
    }
```

**Affected source code:**

- [Fed.sol:105](https://github.com/code-423n4/2022-10-inverse/blob/3e81f0f5908ea99b36e6ab72f13488bbfe622183/src/Fed.sol#L105)

## **7. Lack of no reentrant modifier**

The `Market.getEscrow`, `Fed.expansion` and `Fed.contraction` methods do not have the `noReentrant` modifier and make calls to an external contract that can take advantage of and call these methods again, but it seems to fail due to the lack of tokens.

However, if any of the other addresses used their receive event to provide liquidity to the contract, the attacking account could benefit from it.

```diff
-   function expansion(IMarket market, uint amount) public {
+   function expansion(IMarket market, uint amount) public noReentrant {
        ...
    }

-   function contraction(IMarket market, uint amount) public {
+   function contraction(IMarket market, uint amount) public noReentrant {
        ...
    }
```

For example, in `getEscrow` if the `escrow` allows a callback, it could create two scrows, loosing funds if in this callback it will call again `getEscrow`, using for example `deposit`

```js
    function getEscrow(address user) internal returns (IEscrow) {
        if(escrows[user] != IEscrow(address(0))) return escrows[user];
        IEscrow escrow = createEscrow(user);
        escrow.initialize(collateral, user);
        escrows[user] = escrow;
        return escrow;
    }
```

- Bob call `deposit`.
- During the `escrow` initialization it happend a reentrancy and call again `deposit`.
- The first deposit will be loss in the first escrow.

*Please note that current escrows do not allow re-entry, so I decided to use Low*.
**It's always good to change the storage flags before the externals calls.**

**Affected source code:**

- [Fed.sol:86](https://github.com/code-423n4/2022-10-inverse/blob/3e81f0f5908ea99b36e6ab72f13488bbfe622183/src/Fed.sol#L86)
- [Fed.sol:103](https://github.com/code-423n4/2022-10-inverse/blob/3e81f0f5908ea99b36e6ab72f13488bbfe622183/src/Fed.sol#L103)
- [Market.sol:245](https://github.com/code-423n4/2022-10-inverse/blob/3e81f0f5908ea99b36e6ab72f13488bbfe622183/src/Market.sol#L245)

## **8. Lack of checks the integer ranges**

The following methods lack checks on the following integer arguments, you can see the recommendations above.

**Affected source code:**

`_replenishmentPriceBps` is not checked to be != 0 during the `constructor`, nevertheless it's checked in [setReplenishmentPriceBps](https://github.com/code-423n4/2022-10-inverse/blob/3e81f0f5908ea99b36e6ab72f13488bbfe622183/src/DBR.sol#L63)

- [DBR.sol:36](https://github.com/code-423n4/2022-10-inverse/blob/3e81f0f5908ea99b36e6ab72f13488bbfe622183/src/DBR.sol#L36)

`replenishmentIncentiveBps` is not checked to be > 0 during the `constructor`, nevertheless it's checked in [setReplenismentIncentiveBps](https://github.com/code-423n4/2022-10-inverse/blob/3e81f0f5908ea99b36e6ab72f13488bbfe622183/src/Market.sol#L173)

- [Market.sol:76](https://github.com/code-423n4/2022-10-inverse/blob/3e81f0f5908ea99b36e6ab72f13488bbfe622183/src/Market.sol#L76)

## **9. Lack of checks `supportsInterface`**

The `EIP-165` standard helps detect that a smart contract implements the expected logic, prevents human error when configuring smart contract bindings, so it is recommended to check that the received argument is a contract and supports the expected interface.

**Reference:**

- https://eips.ethereum.org/EIPS/eip-165

**Affected source code:**

- [DBR.sol:99](https://github.com/code-423n4/2022-10-inverse/blob/3e81f0f5908ea99b36e6ab72f13488bbfe622183/src/DBR.sol#L99)
- [Market.sol:81-83](https://github.com/code-423n4/2022-10-inverse/blob/3e81f0f5908ea99b36e6ab72f13488bbfe622183/src/Market.sol#L81-L83)
- [Market.sol:118](https://github.com/code-423n4/2022-10-inverse/blob/3e81f0f5908ea99b36e6ab72f13488bbfe622183/src/Market.sol#L118)
- [Market.sol:124](https://github.com/code-423n4/2022-10-inverse/blob/3e81f0f5908ea99b36e6ab72f13488bbfe622183/src/Market.sol#L124)

## **10. Lack of event emit**

The `Market.pauseBorrows`, `Market.setLiquidationFeeBps`, `Market.setLiquidationIncentiveBps`, `Market.setReplenismentIncentiveBps`, `Market.setLiquidationFactorBps`, `Market.setCollateralFactorBps`, `Market.setBorrowController`, `Market.setOracle` methods do not emit an event when the state changes, something that it's very important for dApps and users.

**Affected source code:**

- [Market.sol:118](https://github.com/code-423n4/2022-10-inverse/blob/3e81f0f5908ea99b36e6ab72f13488bbfe622183/src/Market.sol#L118)
- [Market.sol:124](https://github.com/code-423n4/2022-10-inverse/blob/3e81f0f5908ea99b36e6ab72f13488bbfe622183/src/Market.sol#L124)
- [Market.sol:149](https://github.com/code-423n4/2022-10-inverse/blob/3e81f0f5908ea99b36e6ab72f13488bbfe622183/src/Market.sol#L149)
- [Market.sol:161](https://github.com/code-423n4/2022-10-inverse/blob/3e81f0f5908ea99b36e6ab72f13488bbfe622183/src/Market.sol#L161)
- [Market.sol:172](https://github.com/code-423n4/2022-10-inverse/blob/3e81f0f5908ea99b36e6ab72f13488bbfe622183/src/Market.sol#L172)
- [Market.sol:183](https://github.com/code-423n4/2022-10-inverse/blob/3e81f0f5908ea99b36e6ab72f13488bbfe622183/src/Market.sol#L183)
- [Market.sol:194](https://github.com/code-423n4/2022-10-inverse/blob/3e81f0f5908ea99b36e6ab72f13488bbfe622183/src/Market.sol#L194)
- [Market.sol:218](https://github.com/code-423n4/2022-10-inverse/blob/3e81f0f5908ea99b36e6ab72f13488bbfe622183/src/Market.sol#L218)

## **11. Oracle not compatible with tokens of 19 or more decimals**

Keep in mind that the version of solidity used, despite being greater than `0.8`, does not prevent integer overflows during casting, it only does so in mathematical operations.

In the case that `feed.decimals()` returns 18, and the token is more than 18 decimals, the following subtraction will cause an underflow, denying the oracle service.

```javascript
    uint8 feedDecimals = feeds[token].feed.decimals();  // 18 => [ETH/DAI] https://rinkeby.etherscan.io/address/0x74825dbc8bf76cc4e9494d0ecb210f676efa001d#readContract
    uint8 tokenDecimals = feeds[token].tokenDecimals;   // > 18
    uint8 decimals = 36 - feedDecimals - tokenDecimals; // overflow
```

All pairs have 8 decimals except the [ETH](https://docs.chain.link/docs/ethereum-addresses) pairs, so a token with 19 decimals in ETH, will fault.

**Affected source code:**

- [Oracle.sol:87-98](https://github.com/code-423n4/2022-10-inverse/blob/3e81f0f5908ea99b36e6ab72f13488bbfe622183/src/Oracle.sol#L87-L98)

## **12. Wrong visibility**

The method `accrueDueTokens` doesn't check that the call is made by a market, and it's public, it should be changed to internal or private to be more resilient.

```js
require(markets[msg.sender], "Only markets can call onBorrow");
```

**Affected source code:**

- [DBR.sol:284](https://github.com/code-423n4/2022-10-inverse/blob/3e81f0f5908ea99b36e6ab72f13488bbfe622183/src/DBR.sol#L284)

---

# Non critical

## **13. Bad nomenclature**

The interface `IERC20` contains two methdos that are not pressent in the official ERC20, `delegate` and `delegates`, it's recommended to change the name of the contract because not any ERC20 it's valid.

**Affected source code:**

- [GovTokenEscrow.sol:9-10](https://github.com/code-423n4/2022-10-inverse/blob/3e81f0f5908ea99b36e6ab72f13488bbfe622183/src/escrows/GovTokenEscrow.sol#L9-L10)
- [INVEscrow.sol:10-11](https://github.com/code-423n4/2022-10-inverse/blob/3e81f0f5908ea99b36e6ab72f13488bbfe622183/src/escrows/INVEscrow.sol#L10-L11)

## **14. Open TODO**

The code that contains "open todos" reflects that the development is not finished and that the code can change a posteriori, prior release, with or without audit.

**Affected source code:**

> // TODO: Test whether an immutable variable will persist across proxies

- [INVEscrow.sol:35](https://github.com/code-423n4/2022-10-inverse/blob/3e81f0f5908ea99b36e6ab72f13488bbfe622183/src/escrows/INVEscrow.sol#L35)

## **15. Avoid duplicate code**

The `viewPrice` and `getPrice` methods of the `Oracle` contract are very similar, the only difference being the following peace of code:

```js
            if(todaysLow == 0 || normalizedPrice < todaysLow) {
                dailyLows[token][day] = normalizedPrice;
                todaysLow = normalizedPrice;
                emit RecordDailyLow(token, normalizedPrice);
            }
```

It's recommended to reuse the code in order to be more readable and light.

**Affected source code:**

- [Oracle.sol:126-130](https://github.com/code-423n4/2022-10-inverse/blob/3e81f0f5908ea99b36e6ab72f13488bbfe622183/src/Oracle.sol#L126-L130)
- [Oracle.sol:79-103](https://github.com/code-423n4/2022-10-inverse/blob/3e81f0f5908ea99b36e6ab72f13488bbfe622183/src/Oracle.sol#L79-L103)

## **16. Avoid hardcoded values**

It is not good practice to hardcode values, but if you are dealing with addresses much less, these can change between implementations, networks or projects, so it is convenient to remove these values from the source code.

**Affected source code:**

- [Market.sol:44](https://github.com/code-423n4/2022-10-inverse/blob/3e81f0f5908ea99b36e6ab72f13488bbfe622183/src/Market.sol#L44)

It's recommended to create a factor variable for `10000`:

- [Market.sol:74-76](https://github.com/code-423n4/2022-10-inverse/blob/3e81f0f5908ea99b36e6ab72f13488bbfe622183/src/Market.sol#L74-L76)
- [Market.sol:150](https://github.com/code-423n4/2022-10-inverse/blob/3e81f0f5908ea99b36e6ab72f13488bbfe622183/src/Market.sol#L150)
- [Market.sol:162](https://github.com/code-423n4/2022-10-inverse/blob/3e81f0f5908ea99b36e6ab72f13488bbfe622183/src/Market.sol#L162)
- [Market.sol:173](https://github.com/code-423n4/2022-10-inverse/blob/3e81f0f5908ea99b36e6ab72f13488bbfe622183/src/Market.sol#L173)
- [Market.sol:184](https://github.com/code-423n4/2022-10-inverse/blob/3e81f0f5908ea99b36e6ab72f13488bbfe622183/src/Market.sol#L184)
- [Market.sol:195](https://github.com/code-423n4/2022-10-inverse/blob/3e81f0f5908ea99b36e6ab72f13488bbfe622183/src/Market.sol#L195)
- [Market.sol:336](https://github.com/code-423n4/2022-10-inverse/blob/3e81f0f5908ea99b36e6ab72f13488bbfe622183/src/Market.sol#L336)
- [Market.sol:346](https://github.com/code-423n4/2022-10-inverse/blob/3e81f0f5908ea99b36e6ab72f13488bbfe622183/src/Market.sol#L346)
- [Market.sol:360](https://github.com/code-423n4/2022-10-inverse/blob/3e81f0f5908ea99b36e6ab72f13488bbfe622183/src/Market.sol#L360)
- [Market.sol:377](https://github.com/code-423n4/2022-10-inverse/blob/3e81f0f5908ea99b36e6ab72f13488bbfe622183/src/Market.sol#L377)
- [Market.sol:563-564](https://github.com/code-423n4/2022-10-inverse/blob/3e81f0f5908ea99b36e6ab72f13488bbfe622183/src/Market.sol#L563-L564)
- [Market.sol:583](https://github.com/code-423n4/2022-10-inverse/blob/3e81f0f5908ea99b36e6ab72f13488bbfe622183/src/Market.sol#L583)
- [Market.sol:595-606](https://github.com/code-423n4/2022-10-inverse/blob/3e81f0f5908ea99b36e6ab72f13488bbfe622183/src/Market.sol#L595-L606)
