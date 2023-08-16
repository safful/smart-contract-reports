## Tags

- bug
- G (Gas Optimization)
- sponsor confirmed

# [_recordBurn _payer](https://github.com/code-423n4/2021-11-bootfinance-findings/issues/300) 

# Handle

pauliax


# Vulnerability details

## Impact
function _recordBurn does not really need this parameter of address _payer as it is always equal to msg.sender. 

Consider replacing:
function _recordBurn(address _payer, ...
emit Burn(_payer, ...

with:
function _recordBurn(...
emit Burn(msg.sender, ...


