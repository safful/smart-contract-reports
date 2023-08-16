## Tags

- bug
- 1 (Low Risk)
- sponsor confirmed

# [`HybridPool`'s `flashSwap` does not always call callback](https://github.com/code-423n4/2021-09-sushitrident-findings/issues/100) 

# Handle

cmichel


# Vulnerability details

The `HybridPool.flashSwap` function skips the `tridentSwapCallback` callback call if `data.length == 0`.

## Impact
It should never skip the callback, otherwise the `flashSwap` function is useless.
Note that this behavior of the `HybridPool` is not in alignment with the `flashSwap` behavior of all other pools that indeed always call the callback.

## Recommended Mitigation Steps
Always make the call to `ITridentCallee(msg.sender).tridentSwapCallback(data);`, regardless of the `data` variable.

