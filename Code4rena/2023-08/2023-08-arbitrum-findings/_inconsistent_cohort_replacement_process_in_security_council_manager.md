## Tags

- bug
- 2 (Med Risk)
- primary issue
- satisfactory
- selected for report
- sponsor acknowledged
- M-05

# [ Inconsistent Cohort Replacement Process in Security Council Manager](https://github.com/code-423n4/2023-08-arbitrum-findings/issues/82) 

# Lines of code

https://github.com/ArbitrumFoundation/governance/blob/c18de53820c505fc459f766c1b224810eaeaabc5/src/security-council-mgmt/SecurityCouncilManager.sol#L124-L141


# Vulnerability details

The contract `SecurityCouncilManager` in the Arbitrum DAO's security council system has an inconsistency in its cohort replacement process. This inconsistency could potentially lead to a situation where a member is present in both the old and new cohorts simultaneously, contradicting the governance rules set out in the DAO's constitution.
## Impact
The inconsistency in the cohort replacement process could lead to a situation where a member is elected to a new cohort while also remaining in the old cohort during ongoing elections. This can create confusion, undermine the democratic process, and potentially compromise the integrity of Security Council decisions. Such a scenario directly contradicts the DAO's constitution, which requires careful handling of cohort replacement to avoid conflicts.
## Proof of Concept
The Arbitrum DAO's constitution outlines a careful process for replacing cohorts of Security Council members to avoid race conditions and operational conflicts, particularly during ongoing elections. However, the contract implementation lacks the necessary checks to ensure that cohort replacement aligns with ongoing governance activities.

The `replaceCohort` function is responsible for replacing a cohort with a new set of members. In this function, the old cohort is immediately deleted, and the new cohort is added. This process, while ensuring that only the new members are present in the cohort after replacement, does not account for potential ongoing elections or concurrent transactions.
```solidity
function replaceCohort(address[] memory _newCohort, Cohort _cohort)
    external
    onlyRole(COHORT_REPLACER_ROLE)
{
    if (_newCohort.length != cohortSize) {
        revert InvalidNewCohortLength({cohort: _newCohort, cohortSize: cohortSize});
    }

    // Delete the old cohort
    _cohort == Cohort.FIRST ? delete firstCohort : delete secondCohort;

    // Add members of the new cohort
    for (uint256 i = 0; i < _newCohort.length; i++) {
        _addMemberToCohortArray(_newCohort[i], _cohort);
    }

    _scheduleUpdate();
    emit CohortReplaced(_newCohort, _cohort);
}
```
## Tools Used
Manual
## Recommended Mitigation Steps
One possible mitigation is to introduce a mechanism that prevents cohort replacement while an election is ongoing. 


## Assessed type

Invalid Validation