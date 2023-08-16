## Tags

- bug
- 1 (Low Risk)
- sponsor confirmed

# [pendingGovernance and  Governace address can be same](https://github.com/code-423n4/2021-10-badgerdao-findings/issues/74) 

# Handle

JMukesh


# Vulnerability details

## Impact
when pendingGovernance  call acceptPendingGovernance() , governance value get updated  but pendingGovernance  remain same its not updated to address(0)

 governance = pendingGovernance;

due to which pendingGovernace and Governace share same address which should not happen

## Tools Used
manual review

## Recommended Mitigation Steps
update pendingGovernance to address(0)

