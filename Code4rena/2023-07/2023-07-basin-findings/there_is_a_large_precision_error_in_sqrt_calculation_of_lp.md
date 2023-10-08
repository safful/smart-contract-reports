## Tags

- bug
- 2 (Med Risk)
- primary issue
- satisfactory
- selected for report
- sponsor acknowledged
- M-06

# [There is a large precision error in sqrt calculation of lp](https://github.com/code-423n4/2023-07-basin-findings/issues/190) 

# Lines of code

https://github.com/code-423n4/2023-07-basin/blob/9403cf973e95ef7219622dbbe2a08396af90b64c/src/functions/ConstantProduct2.sol#L53


# Vulnerability details

## Impact

Compared with div, there is a larger precision error in calculating lp through sqrt, so there should be a way to check whether there are excess tokens left when adding liquidity.

## Proof of Concept

```solidity
    function testCalcLpTokenSupplyDiff() public {
        uint256[] memory reserves = new uint256[](2);
        reserves[0] = 1e24 + 1e4;
        reserves[1] = 10000;
        uint256 lp1 = this.calcLpTokenSupply(reserves, bytes(""));
        reserves[0] = 1e24;
        reserves[1] = 10000;
        uint256 lp2 = this.calcLpTokenSupply(reserves, bytes(""));
        assert(lp1 == lp2);
    }
```

When reserve[0] is larger relative to reserve[1], the accuracy error is larger, and unlike div, the accuracy error is only 1, the accuracy error of sqrt is larger.
When the user input the imbalance the amount will left excess reserve, searchers will monitor the contract after the excess reserve accumulation to a certain degree, will withdraw them by removeLiquidityImbalanced.

## Tools Used

Manual review

## Recommended Mitigation Steps

Excess reserve tokens should be returned when the user adds liquidity


## Assessed type

Math