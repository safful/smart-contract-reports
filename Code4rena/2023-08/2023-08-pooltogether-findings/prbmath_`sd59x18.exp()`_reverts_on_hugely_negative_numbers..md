## Tags

- bug
- 2 (Med Risk)
- high quality report
- primary issue
- selected for report
- sponsor confirmed
- M-02

# [PRBMATH `SD59x18.exp()` reverts on hugely negative numbers.](https://github.com/code-423n4/2023-08-pooltogether-findings/issues/146) 

# Lines of code

https://github.com/GenerationSoftware/pt-v5-cgda-liquidator/blob/7f95bcacd4a566c2becb98d55c1886cadbaa8897/src/libraries/ContinuousGDA.sol#L34-L36
https://github.com/GenerationSoftware/pt-v5-cgda-liquidator/blob/7f95bcacd4a566c2becb98d55c1886cadbaa8897/src/libraries/ContinuousGDA.sol#L64
https://github.com/GenerationSoftware/pt-v5-cgda-liquidator/blob/7f95bcacd4a566c2becb98d55c1886cadbaa8897/src/libraries/ContinuousGDA.sol#L85-L87
https://github.com/PaulRBerg/prb-math/blob/5959ef59f906d689c2472ed08797872a1cc00644/src/sd59x18/Math.sol#L168-L181


# Vulnerability details

## Impact
`ContinuousGDA.sol` inherits a version of `PRB Math` that contains a vulnerability in the `SD59x18.exp()` function, which can be reverted on hugely negative numbers. `SD59x18.exp()` is used for calculations in `ContinuousGDA.sol#purchasePrice()` , `ContinuousGDA.sol#purchaseAmount()` and `ContinuousGDA.sol#computeK()`. Recently, the creators of the `PRBMath` have acknowledged this situation. Here is the corresponding [link](https://github.com/PaulRBerg/prb-math/issues/200). This issue should be proactively corrected by `PoolTogether` to avoid unexpected results that corrupt the protocol's computation flow.
## Proof of Concept
There are 05 instances of this issue: [see here](https://github.com/GenerationSoftware/pt-v5-cgda-liquidator/blob/7f95bcacd4a566c2becb98d55c1886cadbaa8897/src/libraries/ContinuousGDA.sol#L34-L36)
```
File: ContinuousGDA.sol
34:    topE = topE.exp().sub(ONE);
36:    bottomE = bottomE.exp();
64:    SD59x18 exp = _decayConstant.mul(_timeSinceLastAuctionStart).exp();
85:    SD59x18 eValue = exponent.exp();
87:    SD59x18 denominator = (_decayConstant.mul(_purchaseAmount).div(_emissionRate)).exp().sub(ONE);
```
[Proof of the bug acknowledgment by the creator of the PRBMath](https://github.com/PaulRBerg/prb-math/issues/200)
`SD59x18.exp()` correctly returns 0 for inputs less than (roughly) -41.45e18, however it starts to throw `PRBMath_SD59x18_Exp2_InputTooBig` when the input gets hugely negative. This is because of the unchecked multiplication in `exp()` overflowing into positive values: [see here](https://github.com/PaulRBerg/prb-math/blob/5959ef59f906d689c2472ed08797872a1cc00644/src/sd59x18/Math.sol#L168-L181)
```
function exp(SD59x18 x) pure returns (SD59x18 result) {
    int256 xInt = x.unwrap();

    // This check prevents values greater than 192e18 from being passed to {exp2}.
    if (xInt > uEXP_MAX_INPUT) {
        revert Errors.PRBMath_SD59x18_Exp_InputTooBig(x);
    }

    unchecked {
        // Inline the fixed-point multiplication to save gas.
        int256 doubleUnitProduct = xInt * uLOG2_E;                // <== overflow
        result = exp2(wrap(doubleUnitProduct / uUNIT));
    }
}
```
## Tools Used
Manual Review and [Proof of the bug acknowledgment by the creator of the PRBMath](https://github.com/PaulRBerg/prb-math/issues/200)
## Recommended Mitigation Steps
A potential fix would be to compare the input with the smallest (most negative) number that can be safely multiplied by `uLOG2_E`, and return 0 if it's smaller. Alternatively, `exp()` could return 0 for inputs smaller than -41.45e18, which are expected to be truncated to zero by `exp2()` anyway.


## Assessed type

Math