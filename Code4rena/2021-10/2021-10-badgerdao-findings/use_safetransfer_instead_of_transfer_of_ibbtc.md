## Tags

- bug
- 1 (Low Risk)
- sponsor confirmed

# [use safeTransfer instead of transfer of ibbtc](https://github.com/code-423n4/2021-10-badgerdao-findings/issues/11) 

# Handle

pants


# Vulnerability details

ibbtc is ERC20Upgradeable. Not all ERC20 contracts supports "blind" transfer method - i.e transfer that you can ignore the return value. You should either check the return value or use openzeppilin safeTransfer 

