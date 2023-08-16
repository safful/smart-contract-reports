## Tags

- bug
- 0 (Non-critical)
- sponsor confirmed

# [Dao.sol: Return votes > consensus](https://github.com/code-423n4/2021-07-spartan-findings/issues/52) 

# Handle

hickuphh3


# Vulnerability details

### Impact

The `hasMajority()`, `hasQuorum()` and `hasMinority()` functions contains the following implementation:

```jsx
if(votes > consensus){
	return true;
} else {
	return false;
}
```

This can be reduced to `return (votes > consensus);`

