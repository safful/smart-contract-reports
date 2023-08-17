## Tags

- bug
- 2 (Med Risk)
- primary issue
- selected for report
- edited-by-warden
- M-01

# [Semi-erroneous Median Value](https://github.com/code-423n4/2022-11-paraspace-findings/issues/38) 

# Lines of code

https://github.com/code-423n4/2022-11-paraspace/blob/main/paraspace-core/contracts/misc/NFTFloorOracle.sol#L429


# Vulnerability details

## Impact
In `NFTFloorOracle.sol`, `_combine()` returns a validated `medianPrice` on line 429 after having `validPriceList` sorted in ascending order.

It will return a correct median as along as the array entails an odd number of valid prices. However, if there were an even number of valid prices, the median was supposed to be the average of the two middle values according to the median formula below:

### Median Formula

![median formula](https://www.gstatic.com/education/formulas2/472522532/en/median_formula.svg)

![x](https://www.gstatic.com/education/formulas2/472522532/en/median_formula_median_formula_var_1.svg)   =	ordered list of values in data set
![n](https://www.gstatic.com/education/formulas2/472522532/en/median_formula_median_formula_var_2.svg)    =	 number of values in data set

The impact could be significant in edge cases and affect all function calls dependent on the finalized `twap`.

## Proof of Concept
Let's assume there are four valid ether prices each of which is 1.5 times more than the previous one:

validPriceList = [1000, 1500, 2250, 3375]

Instead of returning (1500 + 2250) / 2 = 1875, 2250 is returned, incurring a 20% increment or 120 price deviation.

## Tools Used
Manual Inspection

## Recommended Mitigation Steps
Consider having line 429 refactored as follows:

```
        if (validNum % 2 != 0) {
            return (true, validPriceList[validNum / 2]);
        }
        else
            return (true, (validPriceList[validNum / 2] + validPriceList[(validNum / 2) - 1]) / 2); 
```