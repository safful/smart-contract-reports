## Tags

- bug
- 1 (Low Risk)
- resolved
- sponsor confirmed

# [Assert instead require to validate user inputs](https://github.com/code-423n4/2022-01-insure-findings/issues/21) 

# Handle

robee


# Vulnerability details

From solidity docs: Properly functioning code should never reach a failing assert statement; if this happens there is a bug in your contract which you should fix.
With assert the user pays the gas and with require it doesn't. The ETH network gas isn't cheap and users can see it as a scam. 
You have reachable asserts in the following locations (which should be replaced by require / are mistakenly left from development phase):

        InsureDAOERC20.sol : reachable assert in line 31
        Vault.sol : reachable assert in line 167


