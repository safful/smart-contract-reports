## Tags

- bug
- G (Gas Optimization)
- sponsor acknowledged
- sponsor confirmed

# [`tokens[i]` can be memorized](https://github.com/code-423n4/2021-09-yaxis-findings/issues/143) 

# Handle

0xsanson


# Vulnerability details

## Impact
In `StablesConverter.convert` there are multiple storage reads of `tokens[i]` that add up gas. Consider saving the variable in memory.


## Tools Used
editor

## Recommended Mitigation Steps
Rewrite 
```js
IERC20 _token;    // add this
for (uint8 i = 0; i < 3; i++) {
	_token = tokens[i];  // add this
    //if (_output == address(tokens[i])) {
    if (_output == address(_token)) {
        //uint256 _before = tokens[i].balanceOf(address(this));
        uint256 _before = _token.balanceOf(address(this));
        stableSwap3Pool.remove_liquidity_one_coin(
            _inputAmount,
            i,
            _estimatedOutput
        );
        //uint256 _after = tokens[i].balanceOf(address(this));
        uint256 _after = _token.balanceOf(address(this));
        _outputAmount = _after.sub(_before);
        //tokens[i].safeTransfer(msg.sender, _outputAmount);
        _token.safeTransfer(msg.sender, _outputAmount);
        return _outputAmount;
    }
}
```

