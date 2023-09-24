## Tags

- bug
- 2 (Med Risk)
- low quality report
- primary issue
- satisfactory
- selected for report
- M-01

# [`SecurityCouncilMemberElectionGovernor` Owner Can Change `votingPeriod` During an Active Election](https://github.com/code-423n4/2023-08-arbitrum-findings/issues/206) 

# Lines of code

https://github.com/ArbitrumFoundation/governance/blob/c18de53820c505fc459f766c1b224810eaeaabc5/src/security-council-mgmt/governors/SecurityCouncilMemberElectionGovernor.sol#L103-L110
https://github.com/ArbitrumFoundation/governance/blob/c18de53820c505fc459f766c1b224810eaeaabc5/src/security-council-mgmt/governors/modules/SecurityCouncilMemberElectionGovernorCountingUpgradeable.sol#L77-L84


# Vulnerability details

## Impact

- In `SecurityCouncilMemberElectionGovernor` contract : `relay` function enables the contract owner from making calls to any contract address.

- And in `SecurityCouncilMemberElectionGovernorCountingUpgradeable` contract: `setFullWeightDuration` can be accessed only by invoking it from `SecurityCouncilMemberElectionGovernor` which is possible only via `relay` function.

- So the owner can set a new value for `fullWeightDuration` that is used to determine the deadline after which the voting weight will linearly decrease.

- But when setting it; there's no check if there's a current active proposal.

- This makes the voting unfair and the results unreliable as the owner can control the voting power during the election; as increasing the voting power of late voters if `fullWeightDuration` is set to a higher value during active election.

## Proof of Concept

- Code:

  [SecurityCouncilMemberElectionGovernor contract/relay function](https://github.com/ArbitrumFoundation/governance/blob/c18de53820c505fc459f766c1b224810eaeaabc5/src/security-council-mgmt/governors/SecurityCouncilMemberElectionGovernor.sol#L103-L110)

  ```solidity
  File:governance/src/security-council-mgmt/governors/SecurityCouncilMemberElectionGovernor.sol
  Line 103-110:
      function relay(address target, uint256 value, bytes calldata data)
          external
          virtual
          override
          onlyOwner
      {
          AddressUpgradeable.functionCallWithValue(target, data, value);
      }
  ```

  [SecurityCouncilMemberElectionGovernorCountingUpgradeable contract/setFullWeightDuration function](https://github.com/ArbitrumFoundation/governance/blob/c18de53820c505fc459f766c1b224810eaeaabc5/src/security-council-mgmt/governors/modules/SecurityCouncilMemberElectionGovernorCountingUpgradeable.sol#L77-L84)

  ```solidity
  File: governance/src/security-council-mgmt/governors/modules/SecurityCouncilMemberElectionGovernorCountingUpgradeable.sol
  Line 77-84:
     function setFullWeightDuration(uint256 newFullWeightDuration) public onlyGovernance {
        if (newFullWeightDuration > votingPeriod()) {
            revert FullWeightDurationGreaterThanVotingPeriod(newFullWeightDuration, votingPeriod());
        }

        fullWeightDuration = newFullWeightDuration;
        emit FullWeightDurationSet(newFullWeightDuration);
    }
  ```

- Foundry PoC:

1. `testSetVotingPeriodDuringActiveProposal()` test is added to `SecurityCouncilMemberElectionGovernorTest.t.sol` file; where the relay function is invoked by the contract owner to change the `fullWeightDuration` during an active proposal:

   ```solidity
   function testSetVotingPeriodDuringActiveProposal() public {
    //1. initiate a proposal
    _propose(0);
    //2. change fullWeightDuration while the proposal is still active
    assertEq(governor.votingPeriod(), initParams.votingPeriod);
    vm.prank(initParams.owner);
    governor.relay(
      address(governor),
      0,
      abi.encodeWithSelector(governor.setVotingPeriod.selector, 121_212)
    );
    assertEq(governor.votingPeriod(), 121_212);
   }
   ```

2. Test result:

   ```bash
   $ forge test --match-test testSetVotingPeriodDuringActiveProposal
   [PASS] testSetVotingPeriodDuringActiveProposal() (gas: 118129)
   Test result: ok. 1 passed; 0 failed; 0 skipped; finished in 4.51ms
   Ran 1 test suites: 1 tests passed, 0 failed, 0 skipped (1 total tests)
   ```

## Tools Used

Manual Testing & Foundry.

## Recommended Mitigation Steps

Enable setting a new value for `fullWeightDuration` only if there's no active election.


## Assessed type

Governance