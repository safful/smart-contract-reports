## Tags

- bug
- G (Gas Optimization)
- sponsor confirmed

# [Public functions to external](https://github.com/code-423n4/2021-11-fei-findings/issues/27) 

# Handle

robee


# Vulnerability details

The following functions could be set external to save gas and improve code quality. 
External call cost is less expensive than of public functions. 

        The function setExpirationBlock in PegExchanger.sol could be set external
        The function isEnabled in PegExchanger.sol could be set external
        The function party0Accept in PegExchanger.sol could be set external
        The function isExpired in PegExchanger.sol could be set external
        The function party1Accept in PegExchanger.sol could be set external
        The function exchange in PegExchanger.sol could be set external
        The function isEnabled in TRIBERagequit.sol could be set external
        The function ngmi in TRIBERagequit.sol could be set external
        The function requery in TRIBERagequit.sol could be set external
        The function party0Accept in TRIBERagequit.sol could be set external
        The function isExpired in TRIBERagequit.sol could be set external
        The function party1Accept in TRIBERagequit.sol could be set external
        The function recalculate in TRIBERagequit.sol could be set external



