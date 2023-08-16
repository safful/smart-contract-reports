## Tags

- bug
- G (Gas Optimization)
- sponsor confirmed

# [Use bytes32 instead of string when possible](https://github.com/code-423n4/2021-11-bootfinance-findings/issues/176) 

# Handle

pants


# Vulnerability details

If data can fit into 32 bytes, then you should use bytes32 datatype rather than bytes or strings as it is much cheaper in solidity. Basically, Any fixed size variable in solidity is cheaper than variable size. On the MarketPlace.sol contract, string memory variable can be replaced with bytes32 array. That will save gas on the contract.

An example is revert messages. For example look at line 32 of PublicSale.sol.

