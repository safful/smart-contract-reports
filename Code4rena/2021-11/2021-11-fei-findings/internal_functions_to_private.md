## Tags

- bug
- G (Gas Optimization)
- sponsor confirmed

# [Internal functions to private](https://github.com/code-423n4/2021-11-fei-findings/issues/26) 

# Handle

robee


# Vulnerability details

The following functions could be set private to save gas and improve code quality:
        The function takeFrom in PegExchanger.sol could be set internal
        The function giveTo in PegExchanger.sol could be set internal
        The function verifyProof in TRIBERagequit.sol could be set internal
        The function takeFrom in TRIBERagequit.sol could be set internal
        The function processProof in TRIBERagequit.sol could be set internal
        The function giveTo in TRIBERagequit.sol could be set internal
        The function _startCountdown in TRIBERagequit.sol could be set internal

