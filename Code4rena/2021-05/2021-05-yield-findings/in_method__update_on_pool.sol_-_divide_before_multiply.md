## Tags

- bug
- 1 (Low Risk)
- sponsor confirmed

# [In method _update on Pool.sol - Divide before multiply](https://github.com/code-423n4/2021-05-yield-findings/issues/61) 

# Handle

a_delamo


# Vulnerability details

## Impact

In the Pool.sol contract there is the following code:

```
function _update(
        uint128 baseBalance,
        uint128 fyBalance,
        uint112 _baseCached,
        uint112 _fyTokenCached
    ) private {
        ....

            cumulativeBalancesRatio +=
                (scaledFYTokenCached / _baseCached) *
                timeElapsed;
        ....
    }
```

The multiplication should be always placed at the end to avoid miscalculations like the following one:

```
  a = (b/d)*c
  0 = (5/10)*2 


  a = (b * c)/ 2
  1 = (5 * 2)/10

```





