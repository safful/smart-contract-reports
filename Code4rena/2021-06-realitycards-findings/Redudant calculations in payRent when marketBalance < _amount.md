## Tags

- bug
- G (Gas Optimization)
- sponsor confirmed
- resolved

# [Redudant calculations in payRent when marketBalance < _amount](https://github.com/code-423n4/2021-06-realitycards-findings/issues/123) 

# Handle

pauliax


# Vulnerability details

## Impact
  _amount -= (_amount - marketBalance);
is basically the same as:
   _amount = marketBalance;


