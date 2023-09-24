## Tags

- bug
- 2 (Med Risk)
- disagree with severity
- primary issue
- satisfactory
- selected for report
- sponsor acknowledged
- M-04

# [Security Council can undermine any DAO votes to remove a member](https://github.com/code-423n4/2023-08-arbitrum-findings/issues/97) 

# Lines of code

https://github.com/ArbitrumFoundation/governance/blob/c18de53820c505fc459f766c1b224810eaeaabc5/src/security-council-mgmt/SecurityCouncilManager.sol#L183-L190
https://github.com/ArbitrumFoundation/governance/blob/c18de53820c505fc459f766c1b224810eaeaabc5/src/security-council-mgmt/SecurityCouncilManager.sol#L161-L173
https://github.com/ArbitrumFoundation/governance/blob/c18de53820c505fc459f766c1b224810eaeaabc5/src/security-council-mgmt/SecurityCouncilManager.sol#L176-L180
https://github.com/ArbitrumFoundation/governance/blob/c18de53820c505fc459f766c1b224810eaeaabc5/src/security-council-mgmt/SecurityCouncilManager.sol#L143-L159


# Vulnerability details

## Impact
As stated in the updated constitution text:

"Security Council members may only be removed prior to the end of their terms under two conditions:

1. At least 10% of all Votable Tokens have casted votes “in favour” of removal and at least 5/6 (83.33%) of all casted votes are “in favour” of removal; or

2. At least 9 of the Security Council members vote in favour of removal."

However, since the Security Council are the only party with the ability to add a member, they can simply re-add the member that was removed by a DAO vote if they disagree with the vote. This entirely defeats the point of giving the DAO the ability to remove a member of the council mid-term.

## Proof of Concept
A member of the Security Council can be removed mid-term by calling `removeMember` in `SecurityCouncilManager.sol`. This can only be called by addresses that have the `MEMBER_REMOVER_ROLE` role; the `SecurityCouncilMemberRemovalGovernor.sol` contract and the 9 of 12 emergency Security Council:

```
    function removeMember(address _member) external onlyRole(MEMBER_REMOVER_ROLE) {
        if (_member == address(0)) {
            revert ZeroAddress();
        }
        Cohort cohort = _removeMemberFromCohortArray(_member);
        _scheduleUpdate();
        emit MemberRemoved({member: _member, cohort: cohort});
    }
```

The actual member removal is performed by `_removeMemberFromCohortArray`:

```
    function _removeMemberFromCohortArray(address _member) internal returns (Cohort) {
        for (uint256 i = 0; i < 2; i++) {
            address[] storage cohort = i == 0 ? firstCohort : secondCohort;
            for (uint256 j = 0; j < cohort.length; j++) {
                if (_member == cohort[j]) {
                    cohort[j] = cohort[cohort.length - 1];
                    cohort.pop();
                    return i == 0 ? Cohort.FIRST : Cohort.SECOND;
                }
            }
        }
        revert NotAMember({member: _member});
    }
```

As you can see, the member that has been removed isn't tracked/stored anywhere for later querying. So, let's assume that the DAO has successfully voted to remove a member. The `removeMember` method is called when the vote is successful and the update is schedule in the L2 timelock.

The Security Council now have the ability (as the only party with the `MEMBER_ADDER_ROLE` role) to add a new member to fill the relevant cohort on the council by calling `addMember`:

```
    function addMember(address _newMember, Cohort _cohort) external onlyRole(MEMBER_ADDER_ROLE) {
        _addMemberToCohortArray(_newMember, _cohort);
        _scheduleUpdate();
        emit MemberAdded(_newMember, _cohort);
    }
```

Where `_addMemberToCohortArray` looks like:

```
    function _addMemberToCohortArray(address _newMember, Cohort _cohort) internal {
        if (_newMember == address(0)) {
            revert ZeroAddress();
        }
        address[] storage cohort = _cohort == Cohort.FIRST ? firstCohort : secondCohort;
        if (cohort.length == cohortSize) {
            revert CohortFull({cohort: _cohort});
        }
        if (firstCohortIncludes(_newMember)) {
            revert MemberInCohort({member: _newMember, cohort: Cohort.FIRST});
        }
        if (secondCohortIncludes(_newMember)) {
            revert MemberInCohort({member: _newMember, cohort: Cohort.SECOND});
        }

        cohort.push(_newMember);
    }
```

As you can see, this method only checks to ensure the member being added doesn't already exist in either cohort, however there is no validation whether or not this member has been removed before in the current term. As a result, any DAO votes to remove a member can be usurped by the Security Council since they can simply restate the member that was voted to be removed.

## Tools Used
Manual review

## Recommended Mitigation Steps
Any member removals mid-term should be tracked for the lifecycle of the term (6 months) to ensure that the same member can't be re-added. This array should be wiped when a cohort is replaced with a call to `replaceCohort`.


## Assessed type

Invalid Validation