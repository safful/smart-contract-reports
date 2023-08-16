## Tags

- bug
- G (Gas Optimization)
- sponsor confirmed

# [instead of using && in require. just use require multiple time](https://github.com/code-423n4/2022-01-trader-joe-findings/issues/103) 

# Handle

rfa


# Vulnerability details

## Impact
expensive gas

## Proof of Concept
https://github.com/code-423n4/2022-01-trader-joe/blob/main/contracts/RocketJoeFactory.sol#L53-L59
&& operator cost more gas.
## Tools Used

## Recommended Mitigation Steps
use require multiple times instead of &&
```
require(_eventImplementation != address(0), "RJFactory: Addresses can't be null address");
require(_rJoe != address(0),  "RJFactory: Addresses can't be null address");
...

```

