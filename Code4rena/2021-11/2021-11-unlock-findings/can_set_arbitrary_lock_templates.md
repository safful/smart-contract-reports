## Tags

- bug
- 1 (Low Risk)
- sponsor confirmed

# [Can set arbitrary lock templates](https://github.com/code-423n4/2021-11-unlock-findings/issues/158) 

# Handle

cmichel


# Vulnerability details

The `Unlock.setLockTemplate` function sets the default lock tempalte for new lock creations.
However, it does not verify that this lock template is a valid template that was added to `_publicLockVersions` via `addLockTemplate`.

## Impact
A default template with a wrong version number can be set which is incompatible with updating locks through `upgradeLock` (requires `version == currentVersion + 1`).

## Recommended Mitigation Steps
Add new lock templates using `addLockTemplate` first and restrict `setLockTemplate` to only use these templates, not arbitrary code.


