## Tags

- bug
- 2 (Med Risk)
- satisfactory
- selected for report
- sponsor acknowledged
- M-06

# [`Pool._amountToBin()` returns a wrong value when `protocolFeeRatio = 100%`.](https://github.com/code-423n4/2022-12-Stealth-Project-findings/issues/85) 

# Lines of code

https://github.com/code-423n4/2022-12-Stealth-Project/blob/fc8589d7d8c1d8488fd97ccc46e1ff11c8426ac2/maverick-v1/contracts/models/Pool.sol#L549-L551


# Vulnerability details

## Impact
`Pool._amountToBin()` returns a larger value than it should when `protocolFeeRatio = 100%`.

As a result, bin balances might be calculated wrongly.

## Proof of Concept
`delta.deltaInBinInternal` is used to update the bin balances [like this](https://github.com/code-423n4/2022-12-Stealth-Project/blob/fc8589d7d8c1d8488fd97ccc46e1ff11c8426ac2/maverick-v1/contracts/models/Pool.sol#L287-L293).

```solidity
    if (tokenAIn) {
        binBalanceA += delta.deltaInBinInternal.toUint128();
        binBalanceB = Math.clip128(binBalanceB, delta.deltaOutErc.toUint128());
    } else {
        binBalanceB += delta.deltaInBinInternal.toUint128();
        binBalanceA = Math.clip128(binBalanceA, delta.deltaOutErc.toUint128());
    }
```


As we can see [here](https://github.com/code-423n4/2022-12-Stealth-Project/blob/fc8589d7d8c1d8488fd97ccc46e1ff11c8426ac2/maverick-v1/contracts/models/Pool.sol#L608-L611), `_amountToBin()` is used to calculate` delta.deltaInBinInternal` from `deltaInErc` and `feeBasis`.

```solidity
    uint256 feeBasis = Math.mulDiv(binAmountIn, fee, PRBMathUD60x18.SCALE - fee, true);
    delta.deltaInErc = binAmountIn + feeBasis;
    delta.deltaInBinInternal = _amountToBin(delta.deltaInErc, feeBasis);
    delta.excess = swapped ? Math.clip(amountOut, delta.deltaOutErc) : 0;
```

With the above code, the protocol fee should be a portion of `feeBasis` and it is the same as `feeBasis` when `state.protocolFeeRatio = ONE_3_DECIMAL_SCALE`.

But when we check `_amountToBin()`, the actual protocol fee will be `feeBasis + 1` and `delta.deltaInBinInternal` will be less than its possible smallest value(= `binAmountIn`).

```solidity
    function _amountToBin(uint256 deltaInErc, uint256 feeBasis) internal view returns (uint256 amount) { 
        amount = state.protocolFeeRatio != 0 ? Math.clip(deltaInErc, feeBasis.mul(uint256(state.protocolFeeRatio) * PROTOCOL_FEE_SCALE) + 1) : deltaInErc;
    }
```

## Tools Used
Manual Review

## Recommended Mitigation Steps
We should modify `_amountToBin()` like below.

```solidity
    function _amountToBin(uint256 deltaInErc, uint256 feeBasis) internal view returns (uint256 amount) { 
        if (state.protocolFeeRatio == ONE_3_DECIMAL_SCALE)
            return deltaInErc - feeBasis;

        amount = state.protocolFeeRatio != 0 ? Math.clip(deltaInErc, feeBasis.mul(uint256(state.protocolFeeRatio) * PROTOCOL_FEE_SCALE) + 1) : deltaInErc;
    }
```