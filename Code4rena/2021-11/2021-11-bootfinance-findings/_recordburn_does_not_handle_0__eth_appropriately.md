## Tags

- bug
- 1 (Low Risk)
- sponsor confirmed

# [_recordBurn does not handle 0 _eth appropriately](https://github.com/code-423n4/2021-11-bootfinance-findings/issues/274) 

# Handle

pauliax


# Vulnerability details

## Impact
function _recordBurn should validate that _eth > 0. Now it is possible to spam this function with 0 eth burns and fictitiously increase member statistics. 
I have previously reported this issue in a Vader's contest. You can read find details here: https://github.com/code-423n4/2021-04-vader-findings/issues/269

## Recommended Mitigation Steps
Handle case when _eth = 0 in function _recordBurn.

