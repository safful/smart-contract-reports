## Tags

- bug
- 2 (Med Risk)
- judge review requested
- satisfactory
- selected for report
- sponsor disputed
- M-06

# [[Medium - 1] Too few rewards paid over periods in Furnace and StRSR](https://github.com/code-423n4/2023-01-reserve-findings/issues/377) 

# Lines of code

https://github.com/reserve-protocol/protocol/blob/df7ecadc2bae74244ace5e8b39e94bc992903158/contracts/p1/Furnace.sol#L77-L79
https://github.com/reserve-protocol/protocol/blob/df7ecadc2bae74244ace5e8b39e94bc992903158/contracts/p1/StRSR.sol#L509-L512
https://github.com/reserve-protocol/protocol/blob/946d9b101dd77275c6cbfe0bfe9457927bd221a9/contracts/p1/StRSR.sol#L490-L493


# Vulnerability details

## Impact

For two instances in the codebase ([`Furnace`](https://github.com/reserve-protocol/protocol/blob/df7ecadc2bae74244ace5e8b39e94bc992903158/contracts/p1/Furnace.sol#L77-L79) and [`StRSR`](https://github.com/reserve-protocol/protocol/blob/df7ecadc2bae74244ace5e8b39e94bc992903158/contracts/p1/StRSR.sol#L509-L512)), the composed rewards calculation seems to be wrong.
How the rewards are working in these two snippets is that we are first measuring how much `period` or `rewardPeriod` occured since the last payout and calculating in only **one** step the rewards that should be distributed over these periods. In other words, it is composing the ratio over periods.

## Proof of Concept

Taken from the [comments](https://github.com/reserve-protocol/protocol/blob/946d9b101dd77275c6cbfe0bfe9457927bd221a9/contracts/p1/StRSR.sol#L490-L493), we can write the formula of the next rewards payout as:

```
with n = (i+1) > 0, n is the number of periods
rewards{0} = rsrRewards()
payout{i+1} = rewards{i} * payoutRatio
rewards{i+1} = rewards{i} - payout{i+1}
rewards{i+1} = rewards{i} * (1 - payoutRatio)
```

Generalization:
$$u_{i+1} = u_{i} * (1 - r)$$

It's a geometric mean whose growth rate is `(1 - r)`.

Claculation of the sum:

![](https://user-images.githubusercontent.com/51274081/213756676-cbbcf22b-3237-433d-a6e3-9577b6d75474.png)

You can play with the graph [here](https://www.desmos.com/calculator/1fnpsnf8nt).

For a practical example, let's say that our `rsrRewardsAtLastPayout` is 5, with a `rewardRatio` of 0.9
If we had to calculate our compounded rewards, from the formula given by the comments above, we could calculate manually for the first elements. Let's take the sum for n = 3:

$$S = u_{2} + u_{1} + u_{0}$$
$$u_{2} = u_{1} * (1-0.9)$$
$$u_{1} = u_{0} * (1-0.9)$$
$$u_{0} = rsrRewardsAtLastPayout$$

So,

$$S = u_{0} * (1-0.9) * (1-0.9) + u_{0} * (1-0.9) + u_{0}$$

For the values given above, that's

$$S = 5 * 0.1² + 5 * 0.1 + 5$$
$$S = 5.55$$

If we do the same calculation with the sum formula

![](https://user-images.githubusercontent.com/51274081/213756989-03bb092b-a0a3-447a-a0e8-83035559be7b.png)
$$S' = 4.995$$

## Tools Used

Manual inspection

## Recommended Mitigation Steps

Rather than dividing by 1 (1e18 from the Fixed library), divide it by the `ratio`.

```solidity
// Furnace.sol
// Paying out the ratio r, N times, equals paying out the ratio (1 - (1-r)^N) 1 time.
uint192 payoutRatio = FIX_ONE.minus(FIX_ONE.minus(ratio).powu(numPeriods));

uint256 amount = payoutRatio * lastPayoutBal / ratio;
```

```solidity
// StRSR.sol
uint192 payoutRatio = FIX_ONE - FixLib.powu(FIX_ONE - rewardRatio, numPeriods);

// payout: {qRSR} = D18{1} * {qRSR} / r
uint256 payout = (payoutRatio * rsrRewardsAtLastPayout) / rewardRatio;
```