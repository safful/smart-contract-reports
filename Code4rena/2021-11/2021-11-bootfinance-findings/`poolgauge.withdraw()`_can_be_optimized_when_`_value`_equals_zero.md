## Tags

- bug
- G (Gas Optimization)
- sponsor confirmed

# [`PoolGauge.withdraw()` can be optimized when `_value` equals zero](https://github.com/code-423n4/2021-11-bootfinance-findings/issues/47) 

# Handle

pants


# Vulnerability details

The function `PoolGauge.withdraw()` doesn't have to execute these lines of code when `_value` equals zero:
```
_balance: uint256 = self.balanceOf[msg.sender] - _value
_supply: uint256 = self.totalSupply - _value
self.balanceOf[msg.sender] = _balance
self.totalSupply = _supply

self._update_liquidity_limit(msg.sender, _balance, _supply)

assert ERC20(self.lp_token).transfer(msg.sender, _value)
```

## Impact
There is no reason to execute these lines of code if `_value` equals zero because they won't affect the system. An identical optimization is already implemented in `PoolGauge.deposit()`.

## Tool Used
Manual code review.

## Recommended Mitigation Steps
Execute these lines of code only if `_value` doesn't equal zero.

