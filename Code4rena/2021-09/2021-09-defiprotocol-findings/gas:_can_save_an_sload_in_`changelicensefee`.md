## Tags

- bug
- G (Gas Optimization)
- sponsor confirmed

# [Gas: Can save an sload in `changeLicenseFee`](https://github.com/code-423n4/2021-09-defiprotocol-findings/issues/228) 

# Handle

cmichel


# Vulnerability details

The `if`-branch of `Basket.changeLicenseFee` function ensures that `pendingLicenseFee.licenseFee == newLicenseFee` which means setting `licenseFee = newLicenseFee` is equivalent to `licenseFee = pendingLicenseFee.licenseFee` but the former saves an expensive storage load operation.

