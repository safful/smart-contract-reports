## Tags

- bug
- primary issue
- QA (Quality Assurance)
- selected for report

# [QA Report](https://github.com/code-423n4/2022-10-thegraph-findings/issues/230) 



### Non-Critical Issues List
| Number |Issues Details|Context|
|:--:|:-------|:--:|
| [N-01 ]| Testing all functions is best practice | |
| [N-02] |Include ``return parameters`` in _NatSpec comments_|7|
| [N-03] | `0` address check | 6 |
| [N-04] | Use a more recent version of Solidity | All contracts |
| [N-05] | `Require` revert cause should be known | 4 |
| [N-06] | Use `require` instead of `assert` | 1 |
| [N-07] | For modern and more readable code; update import usages | 13 |
| [N-08] | `Function writing` that does not comply with the `Solidity Style Guide` | 7 |
| [N-09] | Implement some type of version counter that will be incremented automatically for contract upgrades| 2 |
| [N-10] | NatSpec is Missing| 4 |
| [N-20] | Omissions in Events| 11 |
| [N-12] | Constant values such as a call to keccak256(), should used to immutable rather than constant|  |
| [N-13] | Floating pragma| All contracts |
| [N-14] | Inconsistent Solidity Versions |  |
| [N-15] | For extended "using-for" usage, use the latest pragma version |  |
| [N-16] | For functions, follow Solidity standard naming conventions | 2 |

Total 16 issues


### Low Risk Issues List
| Number |Issues Details|Context
|:--:|:-------|:--:|
|[L-01]| Add to _blacklist function_ | |
|[L-02]| Using Vulnerable Version of Openzeppelin| 1 |
|[L-03]| A protected initializer function that can be called at most once must be defined| 4 |
|[L-04]| Avoid _shadowing_ `inherited state variables` | 1 |
|[L-05]| NatSpec comments should be increased in contracts | 12 |

Total 5 issues

### [N-01] Testing all functions is best practice 

**Description:**
The test coverage rate of 91%. Testing all functions is best practice in terms of security criteria.
```js
-----------------------------|----------|----------|----------|----------|----------------|
File                         |  % Stmts | % Branch |  % Funcs |  % Lines |Uncovered Lines |
-----------------------------|----------|----------|----------|----------|----------------|
  GNS.sol                    |    88.59 |    82.69 |    91.67 |    88.67 |... 738,741,792 |
 epochs/                     |    85.19 |       50 |    90.91 |    85.19 |                |
  EpochManager.sol           |    85.19 |       50 |    90.91 |    85.19 |   93,95,96,101 |
  Managed.sol                |    94.29 |     87.5 |       95 |    94.87 |        154,164 |
  Pausable.sol               |    86.67 |       75 |      100 |    86.67 |          28,42 |
 libraries/                  |    95.24 |       50 |      100 |    97.67 |                |
  HexStrings.sol             |    93.33 |       50 |      100 |    93.75 |             13 |
  Staking.sol                |    96.97 |    94.74 |    93.33 |    96.98 |... 1,1588,1600 |
  LibFixedMath.sol           |    71.69 |    56.58 |    47.37 |    71.69 |... 403,404,405 |
 statechannels/              |    90.48 |    81.25 |    91.67 |    90.48 |                |
  AllocationExchange.sol     |    92.59 |       85 |     87.5 |    92.59 |        136,137 |
  GRTWithdrawHelper.sol      |    86.67 |       75 |      100 |    86.67 |          72,73 |
 upgrades/                   |    98.25 |    65.38 |    96.97 |    95.77 |                |
  GraphProxy.sol             |      100 |    71.43 |      100 |    96.67 |            200 |
  GraphProxyStorage.sol      |    91.67 |        0 |    85.71 |    89.47 |          62,63 |
-----------------------------|----------|----------|----------|----------|----------------|
All files                    |    91.71 |    81.07 |    90.79 |    91.77 |                |
-----------------------------|----------|----------|----------|----------|----------------|
```


### [N-02] Include ``return parameters`` in _NatSpec comments_

**Context:**
[IRewardsManager.sol](https://github.com/code-423n4/2022-10-thegraph/blob/main/contracts/rewards/IRewardsManager.sol)
[IStaking.sol](https://github.com/code-423n4/2022-10-thegraph/blob/main/contracts/staking/IStaking.sol)
[IGraphToken.sol](https://github.com/code-423n4/2022-10-thegraph/blob/main/contracts/token/IGraphToken.sol)
[GraphTokenGateway.sol#L54](https://github.com/code-423n4/2022-10-thegraph/blob/main/contracts/gateway/GraphTokenGateway.sol#L54)
[IGraphProxy.sol](https://github.com/code-423n4/2022-10-thegraph/blob/main/contracts/upgrades/IGraphProxy.sol)
[IEpochManager.sol](https://github.com/code-423n4/2022-10-thegraph/blob/main/contracts/epochs/IEpochManager.sol)
[IController.sol](https://github.com/code-423n4/2022-10-thegraph/blob/main/contracts/governance/IController.sol)

**Description:**
It is recommended that Solidity contracts are fully annotated using NatSpec for all public interfaces (everything in the ABI). It is clearly stated in the Solidity official documentation. In complex projects such as Defi, the interpretation of all functions and their arguments and returns is important for code readability and auditability.

https://docs.soliditylang.org/en/v0.8.15/natspec-format.html

If Return parameters are declared, you must prefix them with "/// @return".

Some code analysis programs do analysis by reading NatSpec details, if they can't see the "@return" tag, they do incomplete analysis.

**Recommendation:**
Include return parameters in NatSpec comments

Recommendation Code Style:
 ```js
    /// @notice information about what a function does
    /// @param pageId The id of the page to get the URI for.
    /// @return Returns a page's URI if it has been minted 
    function tokenURI(uint256 pageId) public view virtual override returns (string memory) {
        if (pageId == 0 || pageId > currentId) revert("NOT_MINTED");

        return string.concat(BASE_URI, pageId.toString());
    }
```


### [N-03] `0 address` check

**Context:**
[BridgeEscrow.sol#L21](https://github.com/code-423n4/2022-10-thegraph/blob/main/contracts/gateway/BridgeEscrow.sol#L21)
[L2GraphToken.sol#L81](https://github.com/code-423n4/2022-10-thegraph/blob/main/contracts/l2/token/L2GraphToken.sol#L81)
[GraphProxyAdmin.sol#L69](https://github.com/code-423n4/2022-10-thegraph/blob/main/contracts/upgrades/GraphProxyAdmin.sol#L69)
[GraphProxyStorage.sol#L80](https://github.com/code-423n4/2022-10-thegraph/blob/main/contracts/upgrades/GraphProxyStorage.sol#L80)
[GraphProxy.sol#L116](https://github.com/code-423n4/2022-10-thegraph/blob/main/contracts/upgrades/GraphProxy.sol#L116)
[GraphTokenUpgradeable.sol#L133](https://github.com/code-423n4/2022-10-thegraph/blob/main/contracts/l2/token/GraphTokenUpgradeable.sol#L133)

**Description:**
Also check of the address to protect the code from 0x0 address  problem just in case. This is best practice or instead of suggesting that they verify address != 0x0, you could add some good NatSpec comments explaining what is valid and what is invalid and what are the implications of accidentally using an invalid address.

**Recommendation:**
like this;
`if (_treasury == address(0)) revert ADDRESS_ZERO();`


### [N-04] Use a more recent version of Solidity

**Context:**
All contracts

**Recommendation:**
Old version of Solidity is used `0.7.6`, newer version can be used `0.8.17`


### [N-05] Require revert cause should be known

*Context:**
[GraphProxy.sol#L133](https://github.com/code-423n4/2022-10-thegraph/blob/main/contracts/upgrades/GraphProxy.sol#L133)
[GraphProxyAdmin.sol#L34](https://github.com/code-423n4/2022-10-thegraph/blob/main/contracts/upgrades/GraphProxyAdmin.sol#L34)
[GraphProxyAdmin.sol#L47](https://github.com/code-423n4/2022-10-thegraph/blob/main/contracts/upgrades/GraphProxyAdmin.sol#L47)
[GraphProxyAdmin.sol#L59](https://github.com/code-423n4/2022-10-thegraph/blob/main/contracts/upgrades/GraphProxyAdmin.sol#L59)

**Description:**
Vulnerability related to  description is not written to require, it must be written so that the user knows why the process is reverted.

**Recommendation:**
Current Code:
```js
require(sell.order.side == Side.Sell);
```
Recommendation Code:
```js
require(sell.order.side == Side.Sell, "Sell has invalid parameters");
```

### [N-06] Use `require` instead of `assert`

**Context:**
[GraphProxy.sol#L47-L54](https://github.com/code-423n4/2022-10-thegraph/blob/main/contracts/upgrades/GraphProxy.sol#L47-L54)

**Description:**
Assert should not be used except for tests, `require` should be used

Prior to Solidity 0.8.0, pressing a confirm consumes the remainder of the process's available gas instead of returning it, as request()/revert() did.

Assertion() should be avoided even after solidity version 0.8.0, because its documentation states "The Assert function generates an error of type Panic(uint256). Code that works properly should never Panic, even on invalid external input. If this happens, you need to fix it in your contract. there's a mistake".


### [N-07] For modern and more readable code; update import usages

**Context:**
[ICuration.sol](https://github.com/code-423n4/2022-10-thegraph/blob/main/contracts/curation/ICuration.sol), [IGraphCurationToken.sol](https://github.com/code-423n4/2022-10-thegraph/blob/main/contracts/curation/IGraphCurationToken.sol), [BridgeEscrow.sol#L5-L7](https://github.com/code-423n4/2022-10-thegraph/blob/main/contracts/gateway/BridgeEscrow.sol#L5-L7), [L1GraphTokenGateway.sol#L6-L10](https://github.com/code-423n4/2022-10-thegraph/blob/main/contracts/gateway/L1GraphTokenGateway.sol#L6-L10), [Managed.sol#L5-L12](https://github.com/code-423n4/2022-10-thegraph/blob/main/contracts/governance/Managed.sol#L5-L12), [L2GraphTokenGateway.sol#L6-L13](https://github.com/code-423n4/2022-10-thegraph/blob/main/contracts/l2/gateway/L2GraphTokenGateway.sol#L6-L13), [GraphTokenUpgradeable.sol#L5-L10](https://github.com/code-423n4/2022-10-thegraph/blob/main/contracts/l2/token/GraphTokenUpgradeable.sol#L5-L10), [L2GraphToken.sol#L5-L8](https://github.com/code-423n4/2022-10-thegraph/blob/main/contracts/l2/token/L2GraphToken.sol#L5-L8), [IStaking.sol#L6](https://github.com/code-423n4/2022-10-thegraph/blob/main/contracts/staking/IStaking.sol#L6), [IGraphToken.sol#L5](https://github.com/code-423n4/2022-10-thegraph/blob/main/contracts/token/IGraphToken.sol#L5), [GraphProxy.sol#L5-L7](https://github.com/code-423n4/2022-10-thegraph/blob/main/contracts/upgrades/GraphProxy.sol#L5-L7), [GraphProxyAdmin.sol#L5-L8](https://github.com/code-423n4/2022-10-thegraph/blob/main/contracts/upgrades/GraphProxyAdmin.sol#L5-L8), [GraphUpgradeable.sol#L5](https://github.com/code-423n4/2022-10-thegraph/blob/main/contracts/upgrades/GraphUpgradeable.sol#L5)

**Description:**
Our Solidity code is also cleaner in another way that might not be noticeable: the struct Point. We were importing it previously with global import but not using it. The Point struct `polluted the source code` with an unnecessary object we were not using because we did not need it. This was breaking the rule of modularity and modular programming: `only import what you need` Specific imports with curly braces allow us to apply this rule better.

**Recommendation:**
`import {contract1 , contract2} from "filename.sol";`


### [N-08] `Function writing` that does not comply with the `Solidity Style Guide`

**Context:**
[GraphUpgradeable.sol](https://github.com/code-423n4/2022-10-thegraph/blob/main/contracts/upgrades/GraphUpgradeable.sol), [GraphProxyAdmin.sol](https://github.com/code-423n4/2022-10-thegraph/blob/main/contracts/upgrades/GraphProxyAdmin.sol), [GraphProxy.sol](https://github.com/code-423n4/2022-10-thegraph/blob/main/contracts/upgrades/GraphProxy.sol), [Managed.sol](https://github.com/code-423n4/2022-10-thegraph/blob/main/contracts/governance/Managed.sol), [L2GraphTokenGateway.sol](https://github.com/code-423n4/2022-10-thegraph/blob/main/contracts/l2/gateway/L2GraphTokenGateway.sol), [L1GraphTokenGateway.sol](https://github.com/code-423n4/2022-10-thegraph/blob/main/contracts/gateway/L1GraphTokenGateway.sol), [GraphTokenGateway.sol](https://github.com/code-423n4/2022-10-thegraph/blob/main/contracts/gateway/GraphTokenGateway.sol)

**Description:**
Order of Functions; ordering helps readers identify which functions they can call and to find the constructor and fallback definitions easier. But there are contracts in the project that do not comply with this.

https://docs.soliditylang.org/en/v0.8.17/style-guide.html

Functions should be grouped according to their visibility and ordered:

- constructor
- receive function (if exists)
- fallback function (if exists)
- external
- public
- internal
- private
- within a grouping, place the view and pure functions last

### [N-09] Implement some type of version counter that will be incremented automatically for contract upgrades

[GraphProxyAdmin.sol#L77](https://github.com/code-423n4/2022-10-thegraph/blob/main/contracts/upgrades/GraphProxyAdmin.sol#L77)
[GraphProxy.sol#L115](https://github.com/code-423n4/2022-10-thegraph/blob/main/contracts/upgrades/GraphProxy.sol#L115)

**Description:**
As part of the upgradeability of  Proxies , the contract can be upgraded multiple times, where it is a systematic approach to record the version of each upgrade.

```js
 function upgradeTo(address _newImplementation) external ifAdmin {
        _setPendingImplementation(_newImplementation);
    }

```
I suggest implementing some kind of version counter that will be incremented automatically when you upgrade the contract.

**Recommendation:**
```js

uint256 public authorizeUpgradeCounter;

 function upgradeTo(address _newImplementation) external ifAdmin {
        _setPendingImplementation(_newImplementation);
       authorizeUpgradeCounter+=1;

    }
```


### [N-10] NatSpec is Missing

NatSpec is missing for the following function:

[Managed.sol#L43](https://github.com/code-423n4/2022-10-thegraph/blob/main/contracts/governance/Managed.sol#L43)
[Managed.sol#L48](https://github.com/code-423n4/2022-10-thegraph/blob/main/contracts/governance/Managed.sol#L48)
[Managed.sol#L52](https://github.com/code-423n4/2022-10-thegraph/blob/main/contracts/governance/Managed.sol#L52)
[Managed.sol#L56](https://github.com/code-423n4/2022-10-thegraph/blob/main/contracts/governance/Managed.sol#L56)


### [N-11] Omissions in Events

Throughout the codebase, events are generally emitted when sensitive changes are made to the contracts. However, some events are missing important parameters

The events should include the new value and old value where possible:
Events with no old value;;
[L1GraphTokenGateway.sol#L114](https://github.com/code-423n4/2022-10-thegraph/blob/main/contracts/gateway/L1GraphTokenGateway.sol#L114), [L1GraphTokenGateway.sol#L124](https://github.com/code-423n4/2022-10-thegraph/blob/main/contracts/gateway/L1GraphTokenGateway.sol#L124), [L1GraphTokenGateway.sol#L134](https://github.com/code-423n4/2022-10-thegraph/blob/main/contracts/gateway/L1GraphTokenGateway.sol#L134), [L1GraphTokenGateway.sol#L134](https://github.com/code-423n4/2022-10-thegraph/blob/main/contracts/gateway/L1GraphTokenGateway.sol#L144), [L1GraphTokenGateway.sol#L134](https://github.com/code-423n4/2022-10-thegraph/blob/main/contracts/gateway/L1GraphTokenGateway.sol#L156), [L1GraphTokenGateway.sol#L134](https://github.com/code-423n4/2022-10-thegraph/blob/main/contracts/gateway/L1GraphTokenGateway.sol#L168), [L1GraphTokenGateway.sol#L134](https://github.com/code-423n4/2022-10-thegraph/blob/main/contracts/gateway/L1GraphTokenGateway.sol#L134), [L2GraphTokenGateway.sol#L100](https://github.com/code-423n4/2022-10-thegraph/blob/main/contracts/l2/gateway/L2GraphTokenGateway.sol#L100), [L2GraphTokenGateway.sol#L110](https://github.com/code-423n4/2022-10-thegraph/blob/main/contracts/l2/gateway/L2GraphTokenGateway.sol#L110), [L2GraphTokenGateway.sol#L120](https://github.com/code-423n4/2022-10-thegraph/blob/main/contracts/l2/gateway/L2GraphTokenGateway.sol#L120), [Managed.sol#L106](https://github.com/code-423n4/2022-10-thegraph/blob/main/contracts/governance/Managed.sol#L106)


### [N-12] Constant values such as a call to keccak256(), should used to immutable rather than constant

There is a difference between constant variables and immutable variables, and they should each be used in their appropriate contexts.

While it doesn't save any gas because the compiler knows that developers often make this mistake, it's still best to use the right tool for the task at hand.

Constants should be used for literal values written into the code, and immutable variables should be used for expressions, or values calculated in, or passed into the constructor.

There are 5 instances of this issue:
```js
contracts/l2/token/GraphTokenUpgradeable.sol:

    bytes32 private constant DOMAIN_TYPE_HASH =
        keccak256(
            "EIP712Domain(string name,string version,uint256 chainId,address verifyingContract,bytes32 salt)"
        );
    bytes32 private constant DOMAIN_NAME_HASH = keccak256("Graph Token");
    bytes32 private constant DOMAIN_VERSION_HASH = keccak256("0");
    bytes32 private constant DOMAIN_SALT =
        0xe33842a7acd1d5a1d28f25a931703e5605152dc48d64dc4716efdae1f5659591; // Randomly generated salt
    bytes32 private constant PERMIT_TYPEHASH =
        keccak256(
            "Permit(address owner,address spender,uint256 value,uint256 nonce,uint256 deadline)"
        );
```


### [N-13 ] Floating pragma

**Description:**
Contracts should be deployed with the same compiler version and flags that they have been tested with thoroughly. Locking the pragma helps to ensure that contracts do not accidentally get deployed using, for example, an outdated compiler version that might introduce bugs that affect the contract system negatively.
https://swcregistry.io/docs/SWC-103

Floating Pragma List: 
pragma ^0.7.6.   (all contracts)

**Recommendation:**
Lock the pragma version and also consider known bugs (https://github.com/ethereum/solidity/releases) for the compiler version that is chosen.


### [N-14 ] Inconsistent Solidity Versions 

**Description:**
Different Solidity compiler versions are used throughout Src repositories. The following contracts mix versions: 

```js
contracts/governance/IController.sol:
pragma solidity >=0.6.12 <0.8.0;

other contracts:
pragma solidity ^0.7.6;
```

**Recommendation:**
Versions must be consistent with each other.


### [N-15 ] For extended "using-for" usage, use the latest pragma version

**Description:**
https://blog.soliditylang.org/2022/03/16/solidity-0.8.13-release-announcement/


**Recommendation:**
Use solidity pragma version min. 0.8.13


### [N-16 ] For functions, follow Solidity standard naming conventions

**Context:**
[L2GraphTokenGateway.sol#L286](https://github.com/code-423n4/2022-10-thegraph/blob/main/contracts/l2/gateway/L2GraphTokenGateway.sol#L286)
[L1GraphTokenGateway.sol#L290](https://github.com/code-423n4/2022-10-thegraph/blob/main/contracts/gateway/L1GraphTokenGateway.sol#L290)

**Description:**
The above codes don't follow Solidity's standard naming convention,

internal and private functions : the mixedCase format starting with an underscore (_mixedCase starting with an underscore)

public and external functions : only mixedCase

constant variable : UPPER_CASE_WITH_UNDERSCORES (separated by uppercase and underscore)


### [L-01] Add to _blacklist function_

**Description:**
Cryptocurrency mixing service, Tornado Cash, has been blacklisted in the OFAC.
A lot of blockchain companies, token projects, NFT Projects have ```blacklisted``` all Ethereum addresses owned by Tornado Cash listed in the US Treasury Department's sanction against the protocol.
https://home.treasury.gov/policy-issues/financial-sanctions/recent-actions/20220808
In addition, these platforms even ban accounts that have received ETH on their account with Tornadocash.

Some of these Projects;
 USDC (https://www.circle.com/en/usdc)
 Flashbots (https://www.paradigm.xyz/portfolio/flashbots )
 Aave (https://aave.com/)
 Uniswap
 Balancer
 Infura
 Alchemy 
 Opensea
 dYdX 

Details on the subject;
https://twitter.com/bantg/status/1556712790894706688?s=20&t=HUTDTeLikUr6Dv9JdMF7AA

https://cryptopotato.com/defi-protocol-aave-bans-justin-sun-after-he-randomly-received-0-1-eth-from-tornado-cash/

For this reason, every project in the Ethereum network must have a blacklist function, this is a good method to avoid legal problems in the future, apart from the current need.

Transactions from the project by an account funded by Tonadocash or banned by OFAC can lead to legal problems.Especially American citizens may want to get addresses to the blacklist legally, this is not an obligation

If you think that such legal prohibitions should be made directly by validators, this may not be possible:
https://www.paradigm.xyz/2022/09/base-layer-neutrality

```The ban on Tornado Cash makes little sense, because in the end, no one can prevent people from using other mixer smart contracts, or forking the existing ones. It neither hinders cybercrime, nor privacy.```

Here is the most beautiful and close to the project example; Manifold

Manifold Contract
https://etherscan.io/address/0xe4e4003afe3765aca8149a82fc064c0b125b9e5a#code

```js
     modifier nonBlacklistRequired(address extension) {
         require(!_blacklistedExtensions.contains(extension), "Extension blacklisted");
         _;
     }
```
Recommended Mitigation Steps add to Blacklist function and modifier.


### [L-02] Using Vulnerable Version of Openzeppelin

**Context:** 
[package.json](https://github.com/code-423n4/2022-10-thegraph/blob/main/package.json#L27-L28)

**Description:**
The package.json configuration file says that the project is using 3.4.1 – 3.4.2 of OpenZeppelin which has a vulnerability in initializers that call external contracts, 
There is 1 instance of this issue

| Package | Affected versions | Patched versions |
|:--:|:-------:|:------:|
| @openzeppelin/contracts (npm) |>=3.2.0 <4.4.1 | 4.4.1 |
| @openzeppelin/contracts-upgradeable (npm) | >=3.2.0 <4.4.1 | 4.4.1 |


**Recommendation:**
Use patched versions

**Proof Of Concept:**
https://github.com/OpenZeppelin/openzeppelin-contracts/security/advisories/GHSA-9c22-pwxw-p6hx


### [L-03] A protected initializer function that can be called at most once must be defined

**Context:** 
[BridgeEscrow.sol#L20](https://github.com/code-423n4/2022-10-thegraph/blob/main/contracts/gateway/BridgeEscrow.sol#L20)
[L1GraphTokenGateway.sol#L99](https://github.com/code-423n4/2022-10-thegraph/blob/main/contracts/gateway/L1GraphTokenGateway.sol#L99)
[L2GraphTokenGateway.sol#L87](https://github.com/code-423n4/2022-10-thegraph/blob/main/contracts/l2/gateway/L2GraphTokenGateway.sol#L87)
[L2GraphToken.sol#L48](https://github.com/code-423n4/2022-10-thegraph/blob/main/contracts/l2/token/L2GraphToken.sol#L48)

**Description:**
A protected initializer function that can be called at most once must be defined. Admin should be prevented from calling the restartize function, even if it is incorrect

**Recommendation:**
OpenZeppelin recommends that the initializer modifier be applied to constructors
https://forum.openzeppelin.com/t/uupsupgradeable-vulnerability-post-mortem/15680/5


### [L-04]  Avoid _shadowing_ `inherited state variables`

**Context:**
[GraphProxy.sol#L140](https://github.com/code-423n4/2022-10-thegraph/blob/main/contracts/upgrades/GraphProxy.sol#L140)

**Description:**
In `GraphProxy.sol#L140`, there is a local variable named `_pendingImplementation`, but there is a function named `_pendingImplementation()` in the inherited ``` GraphProxyStorage``` with the same name. This use causes compilers to issue warnings, negatively affecting checking and code readability. 

```js
function _acceptUpgrade() internal {
        address _pendingImplementation = _pendingImplementation();
        require(Address.isContract(_pendingImplementation), "Implementation must be a contract");
        require(
            _pendingImplementation != address(0) && msg.sender == _pendingImplementation,
            "Caller must be the pending implementation"
        );        // Codes...
```

**Recommendation:**
Avoid using variables with the same name, including inherited in the same contract, if used, it must be specified in the NatSpec comments.


### [L-05] NatSpec comments should be increased in contracts

**Context:**
[Governed.sol](https://github.com/code-423n4/2022-10-thegraph/blob/main/contracts/governance/Governed.sol), [Managed.sol](https://github.com/code-423n4/2022-10-thegraph/blob/main/contracts/governance/Managed.sol), [GraphTokenGateway.sol](https://github.com/code-423n4/2022-10-thegraph/blob/main/contracts/gateway/GraphTokenGateway.sol), [IGraphCurationToken.sol](https://github.com/code-423n4/2022-10-thegraph/blob/main/contracts/curation/IGraphCurationToken.sol), [IGraphProxy.sol](https://github.com/code-423n4/2022-10-thegraph/blob/main/contracts/upgrades/IGraphProxy.sol), [IEpochManager.sol](https://github.com/code-423n4/2022-10-thegraph/blob/main/contracts/epochs/IEpochManager.sol), [IController.sol](https://github.com/code-423n4/2022-10-thegraph/blob/main/contracts/governance/IController.sol), [IGraphToken.sol](https://github.com/code-423n4/2022-10-thegraph/blob/main/contracts/token/IGraphToken.sol), [IRewardsManager.sol](https://github.com/code-423n4/2022-10-thegraph/blob/main/contracts/rewards/IRewardsManager.sol), [IStakingData.sol](https://github.com/code-423n4/2022-10-thegraph/blob/main/contracts/staking/IStakingData.sol), [ICuration.sol](https://github.com/code-423n4/2022-10-thegraph/blob/main/contracts/curation/ICuration.sol), [IStaking.sol](https://github.com/code-423n4/2022-10-thegraph/blob/main/contracts/staking/IStaking.sol)

**Description:**
It is recommended that Solidity contracts are fully annotated using NatSpec for all public interfaces (everything in the ABI). It is clearly stated in the Solidity official documentation.
In complex projects such as Defi, the interpretation of all functions and their arguments and returns is important for code readability and auditability.
https://docs.soliditylang.org/en/v0.8.15/natspec-format.html

**Recommendation:**
NatSpec comments should be increased in Contracts

```js
Recommendation  Code:
```js
 /**
     * @notice Sets approval for specific withdrawer addresses
     * @dev This function updates the amount of the withdrawApproval mapping value based on the given 3 argument values
     * @param withdrawer_ Withdrawer address from state variable
     * @param token_ instance of ERC20
     * @param amount_ User amount
     * @param permissioned Control of admin for access 
     */
    function setApprovalFor(
        address withdrawer_,
        ERC20 token_,
        uint256 amount_
    ) external permissioned {
        withdrawApproval[withdrawer_][token_] = amount_;
        emit ApprovedForWithdrawal(withdrawer_, token_, amount_);
    }
```


















































