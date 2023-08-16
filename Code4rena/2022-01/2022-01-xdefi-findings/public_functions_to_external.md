## Tags

- bug
- G (Gas Optimization)
- resolved
- sponsor confirmed

# [Public functions to external](https://github.com/code-423n4/2022-01-xdefi-findings/issues/6) 

# Handle

robee


# Vulnerability details


The following functions could be set external to save gas and improve code quality. 
External call cost is less expensive than of public functions. 

        The function withdrawableOf in XDEFIDistribution.sol could be set external
        The function tokenURI in XDEFIDistribution.sol could be set external
        The function getAllTokensForAccount in XDEFIDistributionHelper.sol could be set external
        The function getAllLockedPositionsForAccount in XDEFIDistributionHelper.sol could be set external



