## Tags

- bug
- 0 (Non-critical)
- disagree with severity
- resolved
- sponsor confirmed

# [Require with not comprehensive message](https://github.com/code-423n4/2022-01-xdefi-findings/issues/11) 

# Handle

robee


# Vulnerability details


The following requires has a non comprehensive messages. 
This is very important to add a comprehensive message for any require. Such that the user has enough 
information to know the reason of failure: 

        Solidity file: XDEFIDistribution.sol, In line 227 with Require message: NO_TOKEN
        Solidity file: XDEFIDistribution.sol, In line 232 with Require message: NO_TOKEN



