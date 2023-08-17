## Tags

- bug
- 2 (Med Risk)
- disagree with severity
- selected for report
- M-03

# [Price feed in MCAGRateFeed#getRate is not sufficiently validated and can return stale price](https://github.com/code-423n4/2023-02-kuma-findings/issues/11) 

# Lines of code

https://github.com/code-423n4/2023-02-kuma/blob/3f3d2269fcb3437a9f00ffdd67b5029487435b95/src/kuma-protocol/MCAGRateFeed.sol#L75-L97


# Vulnerability details

## Impact

MCAGRateFeed#getRate may return stale data

## Proof of Concept

        (, int256 answer,,,) = oracle.latestRoundData();

Classic C4 issue. getRate only uses answer but never checks the freshness of the data, which can lead to stale bond pricing data. Stale pricing data can lead to bonds being bought and sold on KUMASwap that otherwise should not be available. This would harm KIBToken holders as KUMASwap may accept bond with too low of a coupon and reduce rewards.

## Tools Used

Manual Review

## Recommended Mitigation Steps

Validate that `updatedAt` has been updated recently enough:

    -   (, int256 answer,,,) = oracle.latestRoundData();
    +   (, int256 answer,,updatedAt,) = oracle.latestRoundData();


    +   if (updatedAt < block.timestamp - MAX_DELAY) {
    +       revert();
    +   }
        
        if (answer < 0) {
            return _MIN_RATE_COUPON;
        }