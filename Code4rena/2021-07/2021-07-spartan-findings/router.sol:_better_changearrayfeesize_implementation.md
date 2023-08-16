## Tags

- bug
- 0 (Non-critical)
- sponsor confirmed

# [Router.sol: Better changeArrayFeeSize implementation](https://github.com/code-423n4/2021-07-spartan-findings/issues/35) 

# Handle

hickuphh3


# Vulnerability details

### Impact

The current `changeArrayFeeSize` implementation deletes the entire feeArray (which is used to calculate normalAverageFee for dividends), resulting in having to rebuild the feeArray again.

It would be better to keep the feeArray as is if the `_size` is greater than the current feeArrayLength, or trim it otherwise, so that the calculation `normalAverageFee` has past trade fees to use and is therefore more accurate.

### Recommended Mitigation Steps

```jsx
function changeArrayFeeSize(uint _size) external onlyDAO {
	arrayFeeSize = _size;
  // trim feeArray to match _size
  if (_size < feeArray.length) {
     uint[] memory tempFeeArray = new uint[](_size);
     // copy feeArray for gas optimization
     uint[] memory _feeArray = feeArray;
		 for (uint i = 0; i < _size; i++) {
        tempFeeArray[i] = _feeArray[i];
		 }
     feeArray = tempFeeArray;
	}
  // otherwise, keep feeArray unchanged
}
```

