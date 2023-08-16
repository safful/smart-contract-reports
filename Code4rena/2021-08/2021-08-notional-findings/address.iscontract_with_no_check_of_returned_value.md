## Tags

- bug
- 1 (Low Risk)
- sponsor confirmed

# [Address.isContract with no check of returned value](https://github.com/code-423n4/2021-08-notional-findings/issues/44) 

# Handle

pauliax


# Vulnerability details

## Impact
function activateNotional calls Address.isContract(...) but does not check the returned value, thus making this call pretty much useless:
  Address.isContract(address(notionalProxy_));

## Recommended Mitigation Steps
Wrap this in a require statement.

