## Tags

- bug
- 0 (Non-critical)
- sponsor confirmed
- resolved

# [questionFinalised is redundant](https://github.com/code-423n4/2021-06-realitycards-findings/issues/111) 

# Handle

pauliax


# Vulnerability details

## Impact
questionFinalised is redundant, it is only set to true or false but never queried or used in any meaningful way.

## Recommended Mitigation Steps
Remove questionFinalised from the codebase.

