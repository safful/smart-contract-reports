## Tags

- bug
- 2 (Med Risk)
- primary issue
- satisfactory
- selected for report
- sponsor confirmed
- M-02

# [The latest malt price can be less than the actual price target and `StabilizerNode.stabilize` will revert](https://github.com/code-423n4/2023-02-malt-findings/issues/36) 

# Lines of code

https://github.com/code-423n4/2023-02-malt/blob/main/contracts/StabilityPod/StabilizerNode.sol#L188
https://github.com/code-423n4/2023-02-malt/blob/main/contracts/StabilityPod/StabilizerNode.sol#L201-L203


# Vulnerability details

## Impact
`StabilizerNode.stabilize` will revert when `latestSample < priceTarget`.

## Proof of Concept
In StabilizerNode.stabilize, when `exchangeRate > priceTarget` and `_msgSender` is not an admin and not whitelisted, it asserts `livePrice > minThreshold`.
And `minThreshold` is calculated as follows:
```
    uint256 priceTarget = maltDataLab.getActualPriceTarget();
```
```
        uint256 latestSample = maltDataLab.maltPriceAverage(0);
        uint256 minThreshold = latestSample -
          (((latestSample - priceTarget) * sampleSlippageBps) / 10000);
```
This code snippet assumes that `latestSample >= priceTarget`. Although `exchangeRate > priceTarget`, `exchangeRate` is the malt average price during `priceAveragePeriod`. But `latestSample` is one of those malt prices. So `latestSample` can be less than `exchangeRate` and `priceTarget`, so `stabilize` will revert in this case.

## Tools Used
Manual Review

## Recommended Mitigation Steps

Use `minThreshold = latestSample + (((priceTarget - latestSample) * sampleSlippageBps) / 10000)` when `priceTarget > latestSample`.