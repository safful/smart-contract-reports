## Tags

- bug
- G (Gas Optimization)
- primary issue
- selected for report

# [Gas Optimizations](https://github.com/code-423n4/2022-10-thegraph-findings/issues/117) 

## Summary

### Gas Optimizations
| |Issue|Instances|Total Gas Saved|
|-|:-|:-:|:-:|
| [G&#x2011;01] | State variables can be packed into fewer storage slots | 1 | - |
| [G&#x2011;02] | Avoid contract existence checks by using low level calls | 3 | 300 |
| [G&#x2011;03] | State variables should be cached in stack variables rather than re-reading them from storage | 10 | 970 |
| [G&#x2011;04] | `internal` functions only called once can be inlined to save gas | 3 | 60 |
| [G&#x2011;05] | `require()`/`revert()` strings longer than 32 bytes cost extra gas | 6 | - |
| [G&#x2011;06] | `keccak256()` should only need to be called on a specific string literal once | 6 | 252 |
| [G&#x2011;07] | Optimize names to save gas | 19 | 418 |
| [G&#x2011;08] | Using `bool`s for storage incurs overhead | 4 | 68400 |
| [G&#x2011;09] | Use a more recent version of solidity | 20 | - |
| [G&#x2011;10] | Use a more recent version of solidity | 3 | - |
| [G&#x2011;11] | Using `> 0` costs more gas than `!= 0` when used on a `uint` in a `require()` statement | 3 | 18 |
| [G&#x2011;12] | Splitting `require()` statements that use `&&` saves gas | 3 | 9 |
| [G&#x2011;13] | Don't compare boolean expressions to boolean literals | 1 | 9 |
| [G&#x2011;14] | Stack variable used as a cheaper cache for a state variable is only used once | 4 | 12 |
| [G&#x2011;15] | `require()` or `revert()` statements that check input arguments should be at the top of the function | 1 | - |
| [G&#x2011;16] | Use custom errors rather than `revert()`/`require()` strings to save gas | 60 | - |
| [G&#x2011;17] | Functions guaranteed to revert when called by normal users can be marked `payable` | 32 | 672 |

Total: 179 instances over 17 issues with **7120 gas** saved

Gas totals use lower bounds of ranges and count two iterations of each `for`-loop. All values above are runtime, not deployment, values; deployment values are listed in the individual issue descriptions



## Gas Optimizations

### [G&#x2011;01]  State variables can be packed into fewer storage slots
If variables occupying the same slot are both written the same function or by the constructor, avoids a separate Gsset (**20000 gas**). Reads of the variables can also be cheaper

*There is 1 instance of this issue:*
```solidity
File: contracts/governance/Pausable.sol

/// @audit Variable ordering with 3 slots instead of the current 4:
///           uint256(32):lastPausePartialTime, uint256(32):lastPauseTime, address(20):pauseGuardian, bool(1):_partialPaused, bool(1):_paused
8:        bool internal _partialPaused;

```
https://github.com/code-423n4/2022-10-thegraph/blob/48e7c7cf641847e07ba07e01176cb17ba8ad6432/contracts/governance/Pausable.sol#L8

### [G&#x2011;02]  Avoid contract existence checks by using low level calls
Prior to 0.8.10 the compiler inserted extra code, including `EXTCODESIZE` (**100 gas**), to check for contract existence for external function calls. In more recent solidity versions, the compiler will not insert these checks if the external call has a return value. Similar behavior can be achieved in earlier versions by using low-level calls, since low level calls never check for contract existence

*There are 3 instances of this issue:*
```solidity
File: contracts/gateway/BridgeEscrow.sol

/// @audit approve()
29:           graphToken().approve(_spender, type(uint256).max);

```
https://github.com/code-423n4/2022-10-thegraph/blob/48e7c7cf641847e07ba07e01176cb17ba8ad6432/contracts/gateway/BridgeEscrow.sol#L29

```solidity
File: contracts/gateway/L1GraphTokenGateway.sol

/// @audit bridge()
77:           IBridge bridge = IInbox(inbox).bridge();

/// @audit l2ToL1Sender()
81:           address l2ToL1Sender = IOutbox(bridge.activeOutbox()).l2ToL1Sender();

```
https://github.com/code-423n4/2022-10-thegraph/blob/48e7c7cf641847e07ba07e01176cb17ba8ad6432/contracts/gateway/L1GraphTokenGateway.sol#L77


### [G&#x2011;03]  State variables should be cached in stack variables rather than re-reading them from storage
The instances below point to the second+ access of a state variable within a function. Caching of a state variable replaces each Gwarmaccess (**100 gas**) with a much cheaper stack read. Other less obvious fixes/optimizations include having local memory caches of state variable structs, or having local caches of state variable contracts/addresses.

*There are 10 instances of this issue:*
```solidity
File: contracts/governance/Governed.sol

/// @audit governor on line 62
65:           emit NewOwnership(oldGovernor, governor);

/// @audit pendingGovernor on line 44
46:           emit NewPendingOwnership(oldPendingGovernor, pendingGovernor);

/// @audit pendingGovernor on line 55
55:               pendingGovernor != address(0) && msg.sender == pendingGovernor,

/// @audit pendingGovernor on line 55
60:           address oldPendingGovernor = pendingGovernor;

/// @audit pendingGovernor on line 60
62:           governor = pendingGovernor;

/// @audit pendingGovernor on line 63
66:           emit NewPendingOwnership(oldPendingGovernor, pendingGovernor);

```
https://github.com/code-423n4/2022-10-thegraph/blob/48e7c7cf641847e07ba07e01176cb17ba8ad6432/contracts/governance/Governed.sol#L65

```solidity
File: contracts/governance/Pausable.sol

/// @audit _partialPaused on line 30
31:           if (_partialPaused) {

/// @audit _partialPaused on line 31
34:           emit PartialPauseChanged(_partialPaused);

/// @audit _paused on line 44
45:           if (_paused) {

/// @audit _paused on line 45
48:           emit PauseChanged(_paused);

```
https://github.com/code-423n4/2022-10-thegraph/blob/48e7c7cf641847e07ba07e01176cb17ba8ad6432/contracts/governance/Pausable.sol#L31

### [G&#x2011;04]  `internal` functions only called once can be inlined to save gas
Not inlining costs **20 to 40 gas** because of two extra `JUMP` instructions and additional stack operations needed for function calls.

*There are 3 instances of this issue:*
```solidity
File: contracts/governance/Managed.sol

43        function _notPartialPaused() internal view {
44:           require(!controller.paused(), "Paused");

52        function _onlyGovernor() internal view {
53:           require(msg.sender == controller.getGovernor(), "Caller must be Controller governor");

56        function _onlyController() internal view {
57:           require(msg.sender == address(controller), "Caller must be Controller");

```
https://github.com/code-423n4/2022-10-thegraph/blob/48e7c7cf641847e07ba07e01176cb17ba8ad6432/contracts/governance/Managed.sol#L43-L44

### [G&#x2011;05]  `require()`/`revert()` strings longer than 32 bytes cost extra gas
Each extra memory word of bytes past the original 32 [incurs an MSTORE](https://gist.github.com/hrkrshnn/ee8fabd532058307229d65dcd5836ddc#consider-having-short-revert-strings) which costs **3 gas**

*There are 6 instances of this issue:*
```solidity
File: contracts/gateway/GraphTokenGateway.sol

19            require(
20                msg.sender == controller.getGovernor() || msg.sender == pauseGuardian,
21                "Only Governor or Guardian can call"
22:           );

```
https://github.com/code-423n4/2022-10-thegraph/blob/48e7c7cf641847e07ba07e01176cb17ba8ad6432/contracts/gateway/GraphTokenGateway.sol#L19-L22

```solidity
File: contracts/governance/Managed.sol

53:           require(msg.sender == controller.getGovernor(), "Caller must be Controller governor");

```
https://github.com/code-423n4/2022-10-thegraph/blob/48e7c7cf641847e07ba07e01176cb17ba8ad6432/contracts/governance/Managed.sol#L53

```solidity
File: contracts/upgrades/GraphProxy.sol

105:          require(_newAdmin != address(0), "Cannot change the admin of a proxy to the zero address");

141:          require(Address.isContract(_pendingImplementation), "Implementation must be a contract");

142           require(
143               _pendingImplementation != address(0) && msg.sender == _pendingImplementation,
144               "Caller must be the pending implementation"
145:          );

```
https://github.com/code-423n4/2022-10-thegraph/blob/48e7c7cf641847e07ba07e01176cb17ba8ad6432/contracts/upgrades/GraphProxy.sol#L105

```solidity
File: contracts/upgrades/GraphUpgradeable.sol

32:           require(msg.sender == _implementation(), "Caller must be the implementation");

```
https://github.com/code-423n4/2022-10-thegraph/blob/48e7c7cf641847e07ba07e01176cb17ba8ad6432/contracts/upgrades/GraphUpgradeable.sol#L32

### [G&#x2011;06]  `keccak256()` should only need to be called on a specific string literal once
It should be saved to an immutable variable, and the variable used instead. If the hash is being used as a part of a function selector, the cast to `bytes4` should also only be done once

*There are 6 instances of this issue:*
```solidity
File: contracts/governance/Managed.sol

114:          return ICuration(_resolveContract(keccak256("Curation")));

122:          return IEpochManager(_resolveContract(keccak256("EpochManager")));

130:          return IRewardsManager(_resolveContract(keccak256("RewardsManager")));

138:          return IStaking(_resolveContract(keccak256("Staking")));

146:          return IGraphToken(_resolveContract(keccak256("GraphToken")));

154:          return ITokenGateway(_resolveContract(keccak256("GraphTokenGateway")));

```
https://github.com/code-423n4/2022-10-thegraph/blob/48e7c7cf641847e07ba07e01176cb17ba8ad6432/contracts/governance/Managed.sol#L114

### [G&#x2011;07]  Optimize names to save gas
`public`/`external` function names and `public` member variable names can be optimized to save gas. See [this](https://gist.github.com/IllIllI000/a5d8b486a8259f9f77891a919febd1a9) link for an example of how it works. Below are the interfaces/abstract contracts that can be optimized so that the most frequently-called functions use the least amount of gas possible during method lookup. Method IDs that have two leading zero bytes can save **128 gas** each during deployment, and renaming functions to have lower method IDs will save **22 gas** per call, [per sorted position shifted](https://medium.com/joyso/solidity-how-does-function-name-affect-gas-consumption-in-smart-contract-47d270d8ac92)

*There are 19 instances of this issue:*
```solidity
File: contracts/curation/ICuration.sol

/// @audit setDefaultReserveRatio(), setMinimumCurationDeposit(), setCurationTaxPercentage(), setCurationTokenMaster(), mint(), burn(), collect(), isCurated(), getCuratorSignal(), getCurationPoolSignal(), getCurationPoolTokens(), tokensToSignal(), signalToTokens(), curationTaxPercentage()
7:    interface ICuration {

```
https://github.com/code-423n4/2022-10-thegraph/blob/48e7c7cf641847e07ba07e01176cb17ba8ad6432/contracts/curation/ICuration.sol#L7

```solidity
File: contracts/curation/IGraphCurationToken.sol

/// @audit initialize(), burnFrom(), mint()
7:    interface IGraphCurationToken is IERC20Upgradeable {

```
https://github.com/code-423n4/2022-10-thegraph/blob/48e7c7cf641847e07ba07e01176cb17ba8ad6432/contracts/curation/IGraphCurationToken.sol#L7

```solidity
File: contracts/epochs/IEpochManager.sol

/// @audit setEpochLength(), runEpoch(), isCurrentEpochRun(), blockNum(), blockHash(), currentEpoch(), currentEpochBlock(), currentEpochBlockSinceStart(), epochsSince(), epochsSinceUpdate()
5:    interface IEpochManager {

```
https://github.com/code-423n4/2022-10-thegraph/blob/48e7c7cf641847e07ba07e01176cb17ba8ad6432/contracts/epochs/IEpochManager.sol#L5

```solidity
File: contracts/gateway/BridgeEscrow.sol

/// @audit initialize(), approveAll(), revokeAll()
15:   contract BridgeEscrow is GraphUpgradeable, Managed {

```
https://github.com/code-423n4/2022-10-thegraph/blob/48e7c7cf641847e07ba07e01176cb17ba8ad6432/contracts/gateway/BridgeEscrow.sol#L15

```solidity
File: contracts/gateway/GraphTokenGateway.sol

/// @audit setPauseGuardian(), setPaused(), paused()
14:   abstract contract GraphTokenGateway is GraphUpgradeable, Pausable, Managed, ITokenGateway {

```
https://github.com/code-423n4/2022-10-thegraph/blob/48e7c7cf641847e07ba07e01176cb17ba8ad6432/contracts/gateway/GraphTokenGateway.sol#L14

```solidity
File: contracts/gateway/ICallhookReceiver.sol

/// @audit onTokenTransfer()
11:   interface ICallhookReceiver {

```
https://github.com/code-423n4/2022-10-thegraph/blob/48e7c7cf641847e07ba07e01176cb17ba8ad6432/contracts/gateway/ICallhookReceiver.sol#L11

```solidity
File: contracts/gateway/L1GraphTokenGateway.sol

/// @audit initialize(), setArbitrumAddresses(), setL2TokenAddress(), setL2CounterpartAddress(), setEscrowAddress(), addToCallhookWhitelist(), removeFromCallhookWhitelist(), getOutboundCalldata()
21:   contract L1GraphTokenGateway is GraphTokenGateway, L1ArbitrumMessenger {

```
https://github.com/code-423n4/2022-10-thegraph/blob/48e7c7cf641847e07ba07e01176cb17ba8ad6432/contracts/gateway/L1GraphTokenGateway.sol#L21

```solidity
File: contracts/governance/IController.sol

/// @audit getGovernor(), setContractProxy(), unsetContractProxy(), updateController(), getContractProxy(), setPartialPaused(), setPaused(), setPauseGuardian(), paused(), partialPaused()
5:    interface IController {

```
https://github.com/code-423n4/2022-10-thegraph/blob/48e7c7cf641847e07ba07e01176cb17ba8ad6432/contracts/governance/IController.sol#L5

```solidity
File: contracts/governance/Managed.sol

/// @audit syncAllContracts()
23:   contract Managed {

```
https://github.com/code-423n4/2022-10-thegraph/blob/48e7c7cf641847e07ba07e01176cb17ba8ad6432/contracts/governance/Managed.sol#L23

```solidity
File: contracts/l2/gateway/L2GraphTokenGateway.sol

/// @audit initialize(), setL2Router(), setL1TokenAddress(), setL1CounterpartAddress(), outboundTransfer(), getOutboundCalldata()
23:   contract L2GraphTokenGateway is GraphTokenGateway, L2ArbitrumMessenger, ReentrancyGuardUpgradeable {

```
https://github.com/code-423n4/2022-10-thegraph/blob/48e7c7cf641847e07ba07e01176cb17ba8ad6432/contracts/l2/gateway/L2GraphTokenGateway.sol#L23

```solidity
File: contracts/l2/token/GraphTokenUpgradeable.sol

/// @audit addMinter(), removeMinter(), renounceMinter(), mint(), isMinter()
28:   contract GraphTokenUpgradeable is GraphUpgradeable, Governed, ERC20BurnableUpgradeable {

```
https://github.com/code-423n4/2022-10-thegraph/blob/48e7c7cf641847e07ba07e01176cb17ba8ad6432/contracts/l2/token/GraphTokenUpgradeable.sol#L28

```solidity
File: contracts/l2/token/L2GraphToken.sol

/// @audit initialize(), setGateway(), setL1Address()
15:   contract L2GraphToken is GraphTokenUpgradeable, IArbToken {

```
https://github.com/code-423n4/2022-10-thegraph/blob/48e7c7cf641847e07ba07e01176cb17ba8ad6432/contracts/l2/token/L2GraphToken.sol#L15

```solidity
File: contracts/rewards/IRewardsManager.sol

/// @audit setIssuanceRate(), setMinimumSubgraphSignal(), setSubgraphAvailabilityOracle(), setDenied(), setDeniedMany(), isDenied(), getNewRewardsPerSignal(), getAccRewardsPerSignal(), getAccRewardsForSubgraph(), getAccRewardsPerAllocatedToken(), getRewards(), updateAccRewardsPerSignal(), takeRewards(), onSubgraphSignalUpdate(), onSubgraphAllocationUpdate()
5:    interface IRewardsManager {

```
https://github.com/code-423n4/2022-10-thegraph/blob/48e7c7cf641847e07ba07e01176cb17ba8ad6432/contracts/rewards/IRewardsManager.sol#L5

```solidity
File: contracts/staking/IStaking.sol

/// @audit setMinimumIndexerStake(), setThawingPeriod(), setCurationPercentage(), setProtocolPercentage(), setChannelDisputeEpochs(), setMaxAllocationEpochs(), setRebateRatio(), setDelegationRatio(), setDelegationParameters(), setDelegationParametersCooldown(), setDelegationUnbondingPeriod(), setDelegationTaxPercentage(), setSlasher(), setAssetHolder(), setOperator(), isOperator(), stake(), stakeTo(), unstake(), slash(), withdraw(), setRewardsDestination(), delegate(), undelegate(), withdrawDelegated(), allocate(), allocateFrom(), closeAllocation(), closeAllocationMany(), closeAndAllocate(), collect(), claim(), claimMany(), hasStake(), getIndexerStakedTokens(), getIndexerCapacity(), getAllocation(), getAllocationState(), isAllocation(), getSubgraphAllocatedTokens(), getDelegation(), isDelegator()
8:    interface IStaking is IStakingData {

```
https://github.com/code-423n4/2022-10-thegraph/blob/48e7c7cf641847e07ba07e01176cb17ba8ad6432/contracts/staking/IStaking.sol#L8

```solidity
File: contracts/token/IGraphToken.sol

/// @audit burn(), burnFrom(), mint(), addMinter(), removeMinter(), renounceMinter(), isMinter()
7:    interface IGraphToken is IERC20 {

```
https://github.com/code-423n4/2022-10-thegraph/blob/48e7c7cf641847e07ba07e01176cb17ba8ad6432/contracts/token/IGraphToken.sol#L7

```solidity
File: contracts/upgrades/GraphProxyAdmin.sol

/// @audit getProxyImplementation(), getProxyPendingImplementation(), getProxyAdmin(), changeProxyAdmin(), upgrade(), acceptProxy(), acceptProxyAndCall()
17:   contract GraphProxyAdmin is Governed {

```
https://github.com/code-423n4/2022-10-thegraph/blob/48e7c7cf641847e07ba07e01176cb17ba8ad6432/contracts/upgrades/GraphProxyAdmin.sol#L17

```solidity
File: contracts/upgrades/GraphProxy.sol

/// @audit implementation(), pendingImplementation(), setAdmin(), upgradeTo(), acceptUpgrade(), acceptUpgradeAndCall()
16:   contract GraphProxy is GraphProxyStorage {

```
https://github.com/code-423n4/2022-10-thegraph/blob/48e7c7cf641847e07ba07e01176cb17ba8ad6432/contracts/upgrades/GraphProxy.sol#L16

```solidity
File: contracts/upgrades/GraphUpgradeable.sol

/// @audit acceptProxy(), acceptProxyAndCall()
11:   contract GraphUpgradeable {

```
https://github.com/code-423n4/2022-10-thegraph/blob/48e7c7cf641847e07ba07e01176cb17ba8ad6432/contracts/upgrades/GraphUpgradeable.sol#L11

```solidity
File: contracts/upgrades/IGraphProxy.sol

/// @audit setAdmin(), implementation(), pendingImplementation(), upgradeTo(), acceptUpgrade(), acceptUpgradeAndCall()
5:    interface IGraphProxy {

```
https://github.com/code-423n4/2022-10-thegraph/blob/48e7c7cf641847e07ba07e01176cb17ba8ad6432/contracts/upgrades/IGraphProxy.sol#L5

### [G&#x2011;08]  Using `bool`s for storage incurs overhead
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
File: contracts/gateway/L1GraphTokenGateway.sol

35:       mapping(address => bool) public callhookWhitelist;

```
https://github.com/code-423n4/2022-10-thegraph/blob/48e7c7cf641847e07ba07e01176cb17ba8ad6432/contracts/gateway/L1GraphTokenGateway.sol#L35

```solidity
File: contracts/governance/Pausable.sol

8:        bool internal _partialPaused;

10:       bool internal _paused;

```
https://github.com/code-423n4/2022-10-thegraph/blob/48e7c7cf641847e07ba07e01176cb17ba8ad6432/contracts/governance/Pausable.sol#L8

```solidity
File: contracts/l2/token/GraphTokenUpgradeable.sol

51:       mapping(address => bool) private _minters;

```
https://github.com/code-423n4/2022-10-thegraph/blob/48e7c7cf641847e07ba07e01176cb17ba8ad6432/contracts/l2/token/GraphTokenUpgradeable.sol#L51

### [G&#x2011;09]  Use a more recent version of solidity
Use a solidity version of at least 0.8.0 to get overflow protection without `SafeMath`
Use a solidity version of at least 0.8.2 to get simple compiler automatic inlining
Use a solidity version of at least 0.8.3 to get better struct packing and cheaper multiple storage reads
Use a solidity version of at least 0.8.4 to get custom errors, which are cheaper at deployment than `revert()/require()` strings
Use a solidity version of at least 0.8.10 to have external calls skip contract existence checks if the external call has a return value

*There are 20 instances of this issue:*
```solidity
File: contracts/curation/ICuration.sol

3:    pragma solidity ^0.7.6;

```
https://github.com/code-423n4/2022-10-thegraph/blob/48e7c7cf641847e07ba07e01176cb17ba8ad6432/contracts/curation/ICuration.sol#L3

```solidity
File: contracts/curation/IGraphCurationToken.sol

3:    pragma solidity ^0.7.6;

```
https://github.com/code-423n4/2022-10-thegraph/blob/48e7c7cf641847e07ba07e01176cb17ba8ad6432/contracts/curation/IGraphCurationToken.sol#L3

```solidity
File: contracts/epochs/IEpochManager.sol

3:    pragma solidity ^0.7.6;

```
https://github.com/code-423n4/2022-10-thegraph/blob/48e7c7cf641847e07ba07e01176cb17ba8ad6432/contracts/epochs/IEpochManager.sol#L3

```solidity
File: contracts/gateway/BridgeEscrow.sol

3:    pragma solidity ^0.7.6;

```
https://github.com/code-423n4/2022-10-thegraph/blob/48e7c7cf641847e07ba07e01176cb17ba8ad6432/contracts/gateway/BridgeEscrow.sol#L3

```solidity
File: contracts/gateway/GraphTokenGateway.sol

3:    pragma solidity ^0.7.6;

```
https://github.com/code-423n4/2022-10-thegraph/blob/48e7c7cf641847e07ba07e01176cb17ba8ad6432/contracts/gateway/GraphTokenGateway.sol#L3

```solidity
File: contracts/gateway/ICallhookReceiver.sol

9:    pragma solidity ^0.7.6;

```
https://github.com/code-423n4/2022-10-thegraph/blob/48e7c7cf641847e07ba07e01176cb17ba8ad6432/contracts/gateway/ICallhookReceiver.sol#L9

```solidity
File: contracts/gateway/L1GraphTokenGateway.sol

3:    pragma solidity ^0.7.6;

```
https://github.com/code-423n4/2022-10-thegraph/blob/48e7c7cf641847e07ba07e01176cb17ba8ad6432/contracts/gateway/L1GraphTokenGateway.sol#L3

```solidity
File: contracts/governance/Governed.sol

3:    pragma solidity ^0.7.6;

```
https://github.com/code-423n4/2022-10-thegraph/blob/48e7c7cf641847e07ba07e01176cb17ba8ad6432/contracts/governance/Governed.sol#L3

```solidity
File: contracts/governance/Managed.sol

3:    pragma solidity ^0.7.6;

```
https://github.com/code-423n4/2022-10-thegraph/blob/48e7c7cf641847e07ba07e01176cb17ba8ad6432/contracts/governance/Managed.sol#L3

```solidity
File: contracts/governance/Pausable.sol

3:    pragma solidity ^0.7.6;

```
https://github.com/code-423n4/2022-10-thegraph/blob/48e7c7cf641847e07ba07e01176cb17ba8ad6432/contracts/governance/Pausable.sol#L3

```solidity
File: contracts/l2/gateway/L2GraphTokenGateway.sol

3:    pragma solidity ^0.7.6;

```
https://github.com/code-423n4/2022-10-thegraph/blob/48e7c7cf641847e07ba07e01176cb17ba8ad6432/contracts/l2/gateway/L2GraphTokenGateway.sol#L3

```solidity
File: contracts/l2/token/GraphTokenUpgradeable.sol

3:    pragma solidity ^0.7.6;

```
https://github.com/code-423n4/2022-10-thegraph/blob/48e7c7cf641847e07ba07e01176cb17ba8ad6432/contracts/l2/token/GraphTokenUpgradeable.sol#L3

```solidity
File: contracts/l2/token/L2GraphToken.sol

3:    pragma solidity ^0.7.6;

```
https://github.com/code-423n4/2022-10-thegraph/blob/48e7c7cf641847e07ba07e01176cb17ba8ad6432/contracts/l2/token/L2GraphToken.sol#L3

```solidity
File: contracts/rewards/IRewardsManager.sol

3:    pragma solidity ^0.7.6;

```
https://github.com/code-423n4/2022-10-thegraph/blob/48e7c7cf641847e07ba07e01176cb17ba8ad6432/contracts/rewards/IRewardsManager.sol#L3

```solidity
File: contracts/token/IGraphToken.sol

3:    pragma solidity ^0.7.6;

```
https://github.com/code-423n4/2022-10-thegraph/blob/48e7c7cf641847e07ba07e01176cb17ba8ad6432/contracts/token/IGraphToken.sol#L3

```solidity
File: contracts/upgrades/GraphProxyAdmin.sol

3:    pragma solidity ^0.7.6;

```
https://github.com/code-423n4/2022-10-thegraph/blob/48e7c7cf641847e07ba07e01176cb17ba8ad6432/contracts/upgrades/GraphProxyAdmin.sol#L3

```solidity
File: contracts/upgrades/GraphProxy.sol

3:    pragma solidity ^0.7.6;

```
https://github.com/code-423n4/2022-10-thegraph/blob/48e7c7cf641847e07ba07e01176cb17ba8ad6432/contracts/upgrades/GraphProxy.sol#L3

```solidity
File: contracts/upgrades/GraphProxyStorage.sol

3:    pragma solidity ^0.7.6;

```
https://github.com/code-423n4/2022-10-thegraph/blob/48e7c7cf641847e07ba07e01176cb17ba8ad6432/contracts/upgrades/GraphProxyStorage.sol#L3

```solidity
File: contracts/upgrades/GraphUpgradeable.sol

3:    pragma solidity ^0.7.6;

```
https://github.com/code-423n4/2022-10-thegraph/blob/48e7c7cf641847e07ba07e01176cb17ba8ad6432/contracts/upgrades/GraphUpgradeable.sol#L3

```solidity
File: contracts/upgrades/IGraphProxy.sol

3:    pragma solidity ^0.7.6;

```
https://github.com/code-423n4/2022-10-thegraph/blob/48e7c7cf641847e07ba07e01176cb17ba8ad6432/contracts/upgrades/IGraphProxy.sol#L3

### [G&#x2011;10]  Use a more recent version of solidity
Use a solidity version of at least 0.8.2 to get simple compiler automatic inlining
Use a solidity version of at least 0.8.3 to get better struct packing and cheaper multiple storage reads
Use a solidity version of at least 0.8.4 to get custom errors, which are cheaper at deployment than `revert()/require()` strings
Use a solidity version of at least 0.8.10 to have external calls skip contract existence checks if the external call has a return value

*There are 3 instances of this issue:*
```solidity
File: contracts/governance/IController.sol

3:    pragma solidity >=0.6.12 <0.8.0;

```
https://github.com/code-423n4/2022-10-thegraph/blob/48e7c7cf641847e07ba07e01176cb17ba8ad6432/contracts/governance/IController.sol#L3

```solidity
File: contracts/staking/IStakingData.sol

3:    pragma solidity >=0.6.12 <0.8.0;

```
https://github.com/code-423n4/2022-10-thegraph/blob/48e7c7cf641847e07ba07e01176cb17ba8ad6432/contracts/staking/IStakingData.sol#L3

```solidity
File: contracts/staking/IStaking.sol

3:    pragma solidity >=0.6.12 <0.8.0;

```
https://github.com/code-423n4/2022-10-thegraph/blob/48e7c7cf641847e07ba07e01176cb17ba8ad6432/contracts/staking/IStaking.sol#L3

### [G&#x2011;11]  Using `> 0` costs more gas than `!= 0` when used on a `uint` in a `require()` statement
This change saves **[6 gas](https://aws1.discourse-cdn.com/business6/uploads/zeppelin/original/2X/3/363a367d6d68851f27d2679d10706cd16d788b96.png)** per instance. The optimization works until solidity version [0.8.13](https://gist.github.com/IllIllI000/bf2c3120f24a69e489f12b3213c06c94) where there is a regression in gas costs.

*There are 3 instances of this issue:*
```solidity
File: contracts/gateway/L1GraphTokenGateway.sol

201:          require(_amount > 0, "INVALID_ZERO_AMOUNT");

217:                  require(maxSubmissionCost > 0, "NO_SUBMISSION_COST");

```
https://github.com/code-423n4/2022-10-thegraph/blob/48e7c7cf641847e07ba07e01176cb17ba8ad6432/contracts/gateway/L1GraphTokenGateway.sol#L201

```solidity
File: contracts/l2/gateway/L2GraphTokenGateway.sol

146:          require(_amount > 0, "INVALID_ZERO_AMOUNT");

```
https://github.com/code-423n4/2022-10-thegraph/blob/48e7c7cf641847e07ba07e01176cb17ba8ad6432/contracts/l2/gateway/L2GraphTokenGateway.sol#L146

### [G&#x2011;12]  Splitting `require()` statements that use `&&` saves gas
See [this issue](https://github.com/code-423n4/2022-01-xdefi-findings/issues/128) which describes the fact that there is a larger deployment gas cost, but with enough runtime calls, the change ends up being cheaper by **3 gas**

*There are 3 instances of this issue:*
```solidity
File: contracts/gateway/L1GraphTokenGateway.sol

142:          require(_escrow != address(0) && Address.isContract(_escrow), "INVALID_ESCROW");

```
https://github.com/code-423n4/2022-10-thegraph/blob/48e7c7cf641847e07ba07e01176cb17ba8ad6432/contracts/gateway/L1GraphTokenGateway.sol#L142

```solidity
File: contracts/governance/Governed.sol

54            require(
55                pendingGovernor != address(0) && msg.sender == pendingGovernor,
56                "Caller must be pending governor"
57:           );

```
https://github.com/code-423n4/2022-10-thegraph/blob/48e7c7cf641847e07ba07e01176cb17ba8ad6432/contracts/governance/Governed.sol#L54-L57

```solidity
File: contracts/upgrades/GraphProxy.sol

142           require(
143               _pendingImplementation != address(0) && msg.sender == _pendingImplementation,
144               "Caller must be the pending implementation"
145:          );

```
https://github.com/code-423n4/2022-10-thegraph/blob/48e7c7cf641847e07ba07e01176cb17ba8ad6432/contracts/upgrades/GraphProxy.sol#L142-L145

### [G&#x2011;13]  Don't compare boolean expressions to boolean literals
`if (<x> == true)` => `if (<x>)`, `if (<x> == false)` => `if (!<x>)`

*There is 1 instance of this issue:*
```solidity
File: contracts/gateway/L1GraphTokenGateway.sol

214:                      extraData.length == 0 || callhookWhitelist[msg.sender] == true,

```
https://github.com/code-423n4/2022-10-thegraph/blob/48e7c7cf641847e07ba07e01176cb17ba8ad6432/contracts/gateway/L1GraphTokenGateway.sol#L214

### [G&#x2011;14]  Stack variable used as a cheaper cache for a state variable is only used once
If the variable is only accessed once, it's cheaper to use the state variable directly that one time, and save the **3 gas** the extra stack assignment would spend

*There are 4 instances of this issue:*
```solidity
File: contracts/governance/Governed.sol

43:           address oldPendingGovernor = pendingGovernor;

59:           address oldGovernor = governor;

60:           address oldPendingGovernor = pendingGovernor;

```
https://github.com/code-423n4/2022-10-thegraph/blob/48e7c7cf641847e07ba07e01176cb17ba8ad6432/contracts/governance/Governed.sol#L43

```solidity
File: contracts/governance/Pausable.sol

56:           address oldPauseGuardian = pauseGuardian;

```
https://github.com/code-423n4/2022-10-thegraph/blob/48e7c7cf641847e07ba07e01176cb17ba8ad6432/contracts/governance/Pausable.sol#L56

### [G&#x2011;15]  `require()` or `revert()` statements that check input arguments should be at the top of the function
Checks that involve constants should come before checks that involve state variables, function calls, and calculations. By doing these checks first, the function is able to revert before wasting a Gcoldsload (**2100 gas***) in a function that may ultimately revert in the unhappy case.

*There is 1 instance of this issue:*
```solidity
File: contracts/gateway/L1GraphTokenGateway.sol

/// @audit expensive op on line 199
201:          require(_amount > 0, "INVALID_ZERO_AMOUNT");

```
https://github.com/code-423n4/2022-10-thegraph/blob/48e7c7cf641847e07ba07e01176cb17ba8ad6432/contracts/gateway/L1GraphTokenGateway.sol#L201

### [G&#x2011;16]  Use custom errors rather than `revert()`/`require()` strings to save gas
Custom errors are available from solidity version 0.8.4. Custom errors save [**~50 gas**](https://gist.github.com/IllIllI000/ad1bd0d29a0101b25e57c293b4b0c746) each time they're hit by [avoiding having to allocate and store the revert string](https://blog.soliditylang.org/2021/04/21/custom-errors/#errors-in-depth). Not defining the strings also save deployment gas

*There are 60 instances of this issue:*
```solidity
File: contracts/gateway/GraphTokenGateway.sol

19            require(
20                msg.sender == controller.getGovernor() || msg.sender == pauseGuardian,
21                "Only Governor or Guardian can call"
22:           );

31:           require(_newPauseGuardian != address(0), "PauseGuardian must be set");

40:           require(!_paused, "Paused (contract)");

```
https://github.com/code-423n4/2022-10-thegraph/blob/48e7c7cf641847e07ba07e01176cb17ba8ad6432/contracts/gateway/GraphTokenGateway.sol#L19-L22

```solidity
File: contracts/gateway/L1GraphTokenGateway.sol

74:           require(inbox != address(0), "INBOX_NOT_SET");

78:           require(msg.sender == address(bridge), "NOT_FROM_BRIDGE");

82:           require(l2ToL1Sender == l2Counterpart, "ONLY_COUNTERPART_GATEWAY");

110:          require(_inbox != address(0), "INVALID_INBOX");

111:          require(_l1Router != address(0), "INVALID_L1_ROUTER");

122:          require(_l2GRT != address(0), "INVALID_L2_GRT");

132:          require(_l2Counterpart != address(0), "INVALID_L2_COUNTERPART");

142:          require(_escrow != address(0) && Address.isContract(_escrow), "INVALID_ESCROW");

153:          require(_newWhitelisted != address(0), "INVALID_ADDRESS");

154:          require(!callhookWhitelist[_newWhitelisted], "ALREADY_WHITELISTED");

165:          require(_notWhitelisted != address(0), "INVALID_ADDRESS");

166:          require(callhookWhitelist[_notWhitelisted], "NOT_WHITELISTED");

200:          require(_l1Token == address(token), "TOKEN_NOT_GRT");

201:          require(_amount > 0, "INVALID_ZERO_AMOUNT");

202:          require(_to != address(0), "INVALID_DESTINATION");

213                   require(
214                       extraData.length == 0 || callhookWhitelist[msg.sender] == true,
215                       "CALL_HOOK_DATA_NOT_ALLOWED"
216:                  );

217:                  require(maxSubmissionCost > 0, "NO_SUBMISSION_COST");

224:                      require(msg.value >= expectedEth, "WRONG_ETH_VALUE");

271:          require(_l1Token == address(token), "TOKEN_NOT_GRT");

275:          require(_amount <= escrowBalance, "BRIDGE_OUT_OF_FUNDS");

```
https://github.com/code-423n4/2022-10-thegraph/blob/48e7c7cf641847e07ba07e01176cb17ba8ad6432/contracts/gateway/L1GraphTokenGateway.sol#L74

```solidity
File: contracts/governance/Governed.sol

24:           require(msg.sender == governor, "Only Governor can call");

41:           require(_newGovernor != address(0), "Governor must be set");

54            require(
55                pendingGovernor != address(0) && msg.sender == pendingGovernor,
56                "Caller must be pending governor"
57:           );

```
https://github.com/code-423n4/2022-10-thegraph/blob/48e7c7cf641847e07ba07e01176cb17ba8ad6432/contracts/governance/Governed.sol#L24

```solidity
File: contracts/governance/Managed.sol

44:           require(!controller.paused(), "Paused");

45:           require(!controller.partialPaused(), "Partial-paused");

49:           require(!controller.paused(), "Paused");

53:           require(msg.sender == controller.getGovernor(), "Caller must be Controller governor");

57:           require(msg.sender == address(controller), "Caller must be Controller");

104:          require(_controller != address(0), "Controller must be set");

```
https://github.com/code-423n4/2022-10-thegraph/blob/48e7c7cf641847e07ba07e01176cb17ba8ad6432/contracts/governance/Managed.sol#L44

```solidity
File: contracts/l2/gateway/L2GraphTokenGateway.sol

69            require(
70                msg.sender == AddressAliasHelper.applyL1ToL2Alias(l1Counterpart),
71                "ONLY_COUNTERPART_GATEWAY"
72:           );

98:           require(_l2Router != address(0), "INVALID_L2_ROUTER");

108:          require(_l1GRT != address(0), "INVALID_L1_GRT");

118:          require(_l1Counterpart != address(0), "INVALID_L1_COUNTERPART");

145:          require(_l1Token == l1GRT, "TOKEN_NOT_GRT");

146:          require(_amount > 0, "INVALID_ZERO_AMOUNT");

147:          require(msg.value == 0, "INVALID_NONZERO_VALUE");

148:          require(_to != address(0), "INVALID_DESTINATION");

153:          require(outboundCalldata.extraData.length == 0, "CALL_HOOK_DATA_NOT_ALLOWED");

233:          require(_l1Token == l1GRT, "TOKEN_NOT_GRT");

234:          require(msg.value == 0, "INVALID_NONZERO_VALUE");

```
https://github.com/code-423n4/2022-10-thegraph/blob/48e7c7cf641847e07ba07e01176cb17ba8ad6432/contracts/l2/gateway/L2GraphTokenGateway.sol#L69-L72

```solidity
File: contracts/l2/token/GraphTokenUpgradeable.sol

60:           require(isMinter(msg.sender), "Only minter can call");

94:           require(_owner == recoveredAddress, "GRT: invalid permit");

95:           require(_deadline == 0 || block.timestamp <= _deadline, "GRT: expired permit");

106:          require(_account != address(0), "INVALID_MINTER");

115:          require(_minters[_account], "NOT_A_MINTER");

123:          require(_minters[msg.sender], "NOT_A_MINTER");

```
https://github.com/code-423n4/2022-10-thegraph/blob/48e7c7cf641847e07ba07e01176cb17ba8ad6432/contracts/l2/token/GraphTokenUpgradeable.sol#L60

```solidity
File: contracts/l2/token/L2GraphToken.sol

36:           require(msg.sender == gateway, "NOT_GATEWAY");

49:           require(_owner != address(0), "Owner must be set");

60:           require(_gw != address(0), "INVALID_GATEWAY");

70:           require(_addr != address(0), "INVALID_L1_ADDRESS");

```
https://github.com/code-423n4/2022-10-thegraph/blob/48e7c7cf641847e07ba07e01176cb17ba8ad6432/contracts/l2/token/L2GraphToken.sol#L36

```solidity
File: contracts/upgrades/GraphProxy.sol

105:          require(_newAdmin != address(0), "Cannot change the admin of a proxy to the zero address");

141:          require(Address.isContract(_pendingImplementation), "Implementation must be a contract");

142           require(
143               _pendingImplementation != address(0) && msg.sender == _pendingImplementation,
144               "Caller must be the pending implementation"
145:          );

157:          require(msg.sender != _admin(), "Cannot fallback to proxy target");

```
https://github.com/code-423n4/2022-10-thegraph/blob/48e7c7cf641847e07ba07e01176cb17ba8ad6432/contracts/upgrades/GraphProxy.sol#L105

```solidity
File: contracts/upgrades/GraphProxyStorage.sol

62:           require(msg.sender == _admin(), "Caller must be admin");

```
https://github.com/code-423n4/2022-10-thegraph/blob/48e7c7cf641847e07ba07e01176cb17ba8ad6432/contracts/upgrades/GraphProxyStorage.sol#L62

```solidity
File: contracts/upgrades/GraphUpgradeable.sol

24:           require(msg.sender == _proxy.admin(), "Caller must be the proxy admin");

32:           require(msg.sender == _implementation(), "Caller must be the implementation");

```
https://github.com/code-423n4/2022-10-thegraph/blob/48e7c7cf641847e07ba07e01176cb17ba8ad6432/contracts/upgrades/GraphUpgradeable.sol#L24

### [G&#x2011;17]  Functions guaranteed to revert when called by normal users can be marked `payable`
If a function modifier such as `onlyOwner` is used, the function will revert if a normal user tries to pay the function. Marking the function as `payable` will lower the gas cost for legitimate callers because the compiler will not include checks for whether a payment was provided. The extra opcodes avoided are 
`CALLVALUE`(2),`DUP1`(3),`ISZERO`(3),`PUSH2`(3),`JUMPI`(10),`PUSH1`(3),`DUP1`(3),`REVERT`(0),`JUMPDEST`(1),`POP`(2), which costs an average of about **21 gas per call** to the function, in addition to the extra deployment cost

*There are 32 instances of this issue:*
```solidity
File: contracts/gateway/BridgeEscrow.sol

20:       function initialize(address _controller) external onlyImpl {

28:       function approveAll(address _spender) external onlyGovernor {

36:       function revokeAll(address _spender) external onlyGovernor {

```
https://github.com/code-423n4/2022-10-thegraph/blob/48e7c7cf641847e07ba07e01176cb17ba8ad6432/contracts/gateway/BridgeEscrow.sol#L20

```solidity
File: contracts/gateway/GraphTokenGateway.sol

30:       function setPauseGuardian(address _newPauseGuardian) external onlyGovernor {

47:       function setPaused(bool _newPaused) external onlyGovernorOrGuardian {

```
https://github.com/code-423n4/2022-10-thegraph/blob/48e7c7cf641847e07ba07e01176cb17ba8ad6432/contracts/gateway/GraphTokenGateway.sol#L30

```solidity
File: contracts/gateway/L1GraphTokenGateway.sol

99:       function initialize(address _controller) external onlyImpl {

109:      function setArbitrumAddresses(address _inbox, address _l1Router) external onlyGovernor {

121:      function setL2TokenAddress(address _l2GRT) external onlyGovernor {

131:      function setL2CounterpartAddress(address _l2Counterpart) external onlyGovernor {

141:      function setEscrowAddress(address _escrow) external onlyGovernor {

152:      function addToCallhookWhitelist(address _newWhitelisted) external onlyGovernor {

164:      function removeFromCallhookWhitelist(address _notWhitelisted) external onlyGovernor {

```
https://github.com/code-423n4/2022-10-thegraph/blob/48e7c7cf641847e07ba07e01176cb17ba8ad6432/contracts/gateway/L1GraphTokenGateway.sol#L99

```solidity
File: contracts/governance/Governed.sol

40:       function transferOwnership(address _newGovernor) external onlyGovernor {

```
https://github.com/code-423n4/2022-10-thegraph/blob/48e7c7cf641847e07ba07e01176cb17ba8ad6432/contracts/governance/Governed.sol#L40

```solidity
File: contracts/governance/Managed.sol

95:       function setController(address _controller) external onlyController {

```
https://github.com/code-423n4/2022-10-thegraph/blob/48e7c7cf641847e07ba07e01176cb17ba8ad6432/contracts/governance/Managed.sol#L95

```solidity
File: contracts/l2/gateway/L2GraphTokenGateway.sol

87:       function initialize(address _controller) external onlyImpl {

97:       function setL2Router(address _l2Router) external onlyGovernor {

107:      function setL1TokenAddress(address _l1GRT) external onlyGovernor {

117:      function setL1CounterpartAddress(address _l1Counterpart) external onlyGovernor {

```
https://github.com/code-423n4/2022-10-thegraph/blob/48e7c7cf641847e07ba07e01176cb17ba8ad6432/contracts/l2/gateway/L2GraphTokenGateway.sol#L87

```solidity
File: contracts/l2/token/GraphTokenUpgradeable.sol

105:      function addMinter(address _account) external onlyGovernor {

114:      function removeMinter(address _account) external onlyGovernor {

132:      function mint(address _to, uint256 _amount) external onlyMinter {

```
https://github.com/code-423n4/2022-10-thegraph/blob/48e7c7cf641847e07ba07e01176cb17ba8ad6432/contracts/l2/token/GraphTokenUpgradeable.sol#L105

```solidity
File: contracts/l2/token/L2GraphToken.sol

48:       function initialize(address _owner) external onlyImpl {

59:       function setGateway(address _gw) external onlyGovernor {

69:       function setL1Address(address _addr) external onlyGovernor {

80:       function bridgeMint(address _account, uint256 _amount) external override onlyGateway {

90:       function bridgeBurn(address _account, uint256 _amount) external override onlyGateway {

```
https://github.com/code-423n4/2022-10-thegraph/blob/48e7c7cf641847e07ba07e01176cb17ba8ad6432/contracts/l2/token/L2GraphToken.sol#L48

```solidity
File: contracts/upgrades/GraphProxyAdmin.sol

68:       function changeProxyAdmin(IGraphProxy _proxy, address _newAdmin) public onlyGovernor {

77:       function upgrade(IGraphProxy _proxy, address _implementation) public onlyGovernor {

86:       function acceptProxy(GraphUpgradeable _implementation, IGraphProxy _proxy) public onlyGovernor {

96        function acceptProxyAndCall(
97            GraphUpgradeable _implementation,
98            IGraphProxy _proxy,
99            bytes calldata _data
100:      ) external onlyGovernor {

```
https://github.com/code-423n4/2022-10-thegraph/blob/48e7c7cf641847e07ba07e01176cb17ba8ad6432/contracts/upgrades/GraphProxyAdmin.sol#L68

```solidity
File: contracts/upgrades/GraphUpgradeable.sol

50:       function acceptProxy(IGraphProxy _proxy) external onlyProxyAdmin(_proxy) {

59        function acceptProxyAndCall(IGraphProxy _proxy, bytes calldata _data)
60            external
61:           onlyProxyAdmin(_proxy)

```
https://github.com/code-423n4/2022-10-thegraph/blob/48e7c7cf641847e07ba07e01176cb17ba8ad6432/contracts/upgrades/GraphUpgradeable.sol#L50

