## Tags

- bug
- G (Gas Optimization)
- sponsor confirmed

# [Pool.sol + Router.sol: Set revenue directly as _fee](https://github.com/code-423n4/2021-07-spartan-findings/issues/56) 

# Handle

hickuphh3


# Vulnerability details

### Impact

```jsx
// Pool.sol: L344-345
map30DPoolRevenue = 0;
map30DPoolRevenue = map30DPoolRevenue+(_fee);

// Router.sol: L317-318
mapAddress_30DayDividends[_pool] = 0;
mapAddress_30DayDividends[_pool] = mapAddress_30DayDividends[_pool] + _fees;
```

can simply be written as

```jsx
map30DPoolRevenue = _fee;

mapAddress_30DayDividends[_pool] = _fees;
```

respectively.

