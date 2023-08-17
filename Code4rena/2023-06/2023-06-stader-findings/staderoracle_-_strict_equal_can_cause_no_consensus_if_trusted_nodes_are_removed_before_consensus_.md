## Tags

- bug
- 2 (Med Risk)
- satisfactory
- selected for report
- sponsor confirmed
- edited-by-warden
- M-05

# [StaderOracle - Strict equal can cause no consensus if trusted nodes are removed before consensus ](https://github.com/code-423n4/2023-06-stader-findings/issues/321) 

# Lines of code

https://github.com/code-423n4/2023-06-stader/blob/main/contracts/StaderOracle.sol#L148
https://github.com/code-423n4/2023-06-stader/blob/main/contracts/StaderOracle.sol#L290


# Vulnerability details

## Impact
```
        if (
            submissionCount == trustedNodesCount / 2 + 1 &&
            _exchangeRate.reportingBlockNumber > exchangeRate.reportingBlockNumber
        ) {
            updateWithInLimitER(
                _exchangeRate.totalETHBalance,
                _exchangeRate.totalETHXSupply,
                _exchangeRate.reportingBlockNumber
            );
        }
```
In `submitExchangeRateData`, consensus is reached if `submissionCount` is strictly equal to desired number. However, trustNodesCount can be decreased and this condition can be never met.

```
        if ((submissionCount == (2 * trustedNodesCount) / 3 + 1)) {
            lastReportedSDPriceData = _sdPriceData;
            lastReportedSDPriceData.sdPriceInETH = getMedianValue(sdPrices);
            delete sdPrices;

```
In `submitSDPrice`, if this case happens, `sdPrices` doesn't get deleted and it will affect the next submission batch's price.

## Proof of Concept
In the above snippet, let's assume trustedNodesCount = 10, submissionCount = 5.
The condition doesn't meet for now (5 != 10/2+1). Then trustedNodesCount decreases to 9.
Next time when a node submits, trustedNodesCount = 9, submissionCount = 6.
Then the condition cannot be met since 6 != 9/2+1.

## Tools Used
Manual

## Recommended Mitigation Steps
Replace strict equal with equal or greater than.
Or replace it with greater than and decrease the right side.

Not sure about adding cooldown for add/remove trusted nodes.






## Assessed type

Other