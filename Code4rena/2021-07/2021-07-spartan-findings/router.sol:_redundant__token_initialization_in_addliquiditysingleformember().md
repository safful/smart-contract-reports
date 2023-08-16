## Tags

- bug
- G (Gas Optimization)
- sponsor confirmed

# [Router.sol: Redundant _token initialization in addLiquiditySingleForMember()](https://github.com/code-423n4/2021-07-spartan-findings/issues/57) 

# Handle

hickuphh3


# Vulnerability details

### Impact

The lines below of the `addLiquiditySingleForMember()` function

```jsx
address _token = token;
if(token == address(0)){_token = WBNB;} // Handle BNB -> WBNB
```

are redundant since `_token` is not used subsequently. Note that `_handleTransferIn()` will perform the handling of native BNB transfers.

### Recommended Mitigation Steps

The mentioned lines above can be removed.

