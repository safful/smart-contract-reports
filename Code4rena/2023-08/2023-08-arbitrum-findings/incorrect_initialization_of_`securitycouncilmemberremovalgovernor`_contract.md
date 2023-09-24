## Tags

- bug
- 2 (Med Risk)
- low quality report
- primary issue
- satisfactory
- selected for report
- M-03

# [Incorrect initialization of `SecurityCouncilMemberRemovalGovernor` contract](https://github.com/code-423n4/2023-08-arbitrum-findings/issues/150) 

# Lines of code

https://github.com/ArbitrumFoundation/governance/blob/c18de53820c505fc459f766c1b224810eaeaabc5/src/security-council-mgmt/governors/SecurityCouncilMemberRemovalGovernor.sol#L70-L76


# Vulnerability details

## Impact
The `SecurityCouncilMemberRemovalGovernor` contract inherits `GovernorUpgradeable` & `EIP712Upgradeable` contracts but does not invoke their individual initialzers during its own initialization. Due to which the state of `GovernorUpgradeable` & `EIP712Upgradeable` contracts remain uninitialized.

```solidity
contract SecurityCouncilMemberRemovalGovernor is
    GovernorUpgradeable,
    GovernorVotesUpgradeable,
    ...
{
    function initialize(
        ...
    ) public initializer {
        __GovernorSettings_init(_votingDelay, _votingPeriod, _proposalThreshold);
        __GovernorCountingSimple_init();
        __GovernorVotes_init(_token);
        __ArbitrumGovernorVotesQuorumFraction_init(_quorumNumerator);
        __GovernorPreventLateQuorum_init(_minPeriodAfterQuorum);
        __ArbitrumGovernorProposalExpirationUpgradeable_init(_proposalExpirationBlocks);
        _transferOwnership(_owner);
        ...
    }
}
```
```solidity
abstract contract GovernorUpgradeable is EIP712Upgradeable, ... {
    function __Governor_init(string memory name_) internal onlyInitializing {
        __EIP712_init_unchained(name_, version());
        __Governor_init_unchained(name_);
    }
    function __Governor_init_unchained(string memory name_) internal onlyInitializing {
        _name = name_;
    }
}
```

Due to this issue:
- In `GovernorUpgradeable` the `_name` storage variable is never initialized and the `name()` function returns an empty string.
- In `EIP712Upgradeable` the `_HASHED_NAME` storage variable is never initialized. This variable is used in the `castVoteBySig` & `castVoteWithReasonAndParamsBySig` functions of `SecurityCouncilMemberRemovalGovernor`. 

The non-initialization of `EIP712Upgradeable` contract breaks the compatibility of `SecurityCouncilMemberRemovalGovernor` contract with the EIP712 standard. 

It should be noted that the other contracts like `SecurityCouncilMemberElectionGovernor` and `SecurityCouncilNomineeElectionGovernor` also inherits the `GovernorUpgradeable` & `EIP712Upgradeable` contracts and initializes them correctly. So the incorrect initialization of `SecurityCouncilMemberRemovalGovernor` also breaks the overall code consistency of protocol.

The `SecurityCouncilMemberElectionGovernor` and `SecurityCouncilNomineeElectionGovernor` contract also force users to do voting only via `castVoteWithReasonAndParams` and `castVoteWithReasonAndParamsBySig` functions. Hence it can be clearly observed that the protocol wants to utilize voting-by-signature feature, which requires EIP712 compatibility.

## Proof of Concept
This test case was added in `test/security-council-mgmt/SecurityCouncilMemberRemovalGovernor.t.sol` and ran using `forge test --mt test_audit`.

```solidity
    function test_audit_notInvoking___Governor_init() public {
        assertEq(scRemovalGov.name(), "");
        assertEq(bytes(scRemovalGov.name()).length, 0);
    }
```

## Tools Used
Foundry

## Recommended Mitigation Steps
Consider initializing the `GovernorUpgradeable` & `EIP712Upgradeable` contracts in `SecurityCouncilMemberRemovalGovernor.initialize` function.
```solidity
    function initialize(
        ...
    ) public initializer {
        __Governor_init("SecurityCouncilMemberRemovalGovernor");
        ...
    }
```



## Assessed type

Other