## Tags

- bug
- 3 (High Risk)
- selected for report

# [Risk users are required to payout if the price of the pegged asset goes higher than underlying](https://github.com/code-423n4/2022-09-y2k-finance-findings/issues/45) 

# Lines of code

https://github.com/code-423n4/2022-09-y2k-finance/blob/ac3e86f07bc2f1f51148d2265cc897e8b494adf7/src/oracles/PegOracle.sol#L46-L83


# Vulnerability details

## Impact

Insurance is to protect the user in case the pegged asset drops significantly below the underlying but risk users are required to payout if the pegged asset is worth more than the underlying

## Proof of Concept

        if (price1 > price2) {
            nowPrice = (price2 * 10000) / price1;
        } else {
            nowPrice = (price1 * 10000) / price2;
        }

The above lines calculates the ratio using the lower of the two prices, which means that in the scenario that the pegged asset is worth more than the underlying, a depeg event will be triggered. This is problematic for two reasons. The first is that many pegged assets are designed to maintain at least the value of the underlying. They put very strong incentives to keep the asset from going below the peg but usually use much looser policies to bring the asset down to the peg, since an upward break from the peg is usually considered benign. The second is that when a pegged asset move above the underlying, the users who are holding the asset are benefiting from the appreciation of the asset; therefor the insurance is not needed.

Because of these two reasons, it is my opinion that sellers would demand a higher premium from buyers as a result of the extra risk introduced by the possibility of having to pay out during an upward depeg. It is also my opinion that these higher premiums would push users seeking insurance to other cheaper products that don't include this risk

## Tools Used

## Recommended Mitigation Steps

The ratio returned should always the ratio of the pegged asset to the underlying (i.e. pegged/underlying)