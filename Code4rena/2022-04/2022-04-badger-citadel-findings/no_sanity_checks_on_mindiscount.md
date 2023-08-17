## Tags

- bug
- QA (Quality Assurance)
- sponsor confirmed

# [no sanity checks on minDiscount](https://github.com/code-423n4/2022-04-badger-citadel-findings/issues/185) 

# Lines of code

https://github.com/code-423n4/2022-04-badger-citadel/blob/main/src/Funding.sol#L356


# Vulnerability details

Unlike maxDiscount, minDiscount is missing some sanity checks:
minDiscount should be smaller than MAX_BPS
minDoscount should be smaller than maxDiscount

