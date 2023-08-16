## Tags

- bug
- 1 (Low Risk)
- sponsor confirmed

# [admin can override investor and stuck its funds in the system ](https://github.com/code-423n4/2021-11-bootfinance-findings/issues/11) 

# Handle

pants


# Vulnerability details

An admin can (by mistake maybe) addInvestor with address that already exists. This way its funds are locked in the system and cannot be withdrawn, even by the admin.



