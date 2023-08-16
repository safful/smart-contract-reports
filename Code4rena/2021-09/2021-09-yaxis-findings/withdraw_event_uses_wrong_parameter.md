## Tags

- bug
- 1 (Low Risk)
- sponsor acknowledged
- sponsor confirmed

# [Withdraw event uses wrong parameter](https://github.com/code-423n4/2021-09-yaxis-findings/issues/122) 

# Handle

cmichel


# Vulnerability details

The `Withdraw` event in `LegacyController.withdraw` emits the `_amount` variable which is the _initial, desired_ amount to withdraw. It should emit the actual withdrawn amount instead, which is transferred in the last `token.balanceOf(address(this))` call.

## Impact
The actual withdrawn amount, which can be lower than `_amount`, is part of the event.
This is usually not what you want (and it can already be decoded from the function argument).

## Recommended Mitigation Steps
Use it or remove it.


