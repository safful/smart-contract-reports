## Tags

- bug
- 3 (High Risk)
- primary issue
- satisfactory
- selected for report
- upgraded by judge
- edited-by-warden
- H-01

# [Incorrect `blocksPerYear` constant in `WhitepaperInterestRateModel`](https://github.com/code-423n4/2023-05-venus-findings/issues/320) 

# Lines of code

https://github.com/code-423n4/2023-05-venus/blob/8be784ed9752b80e6f1b8b781e2e6251748d0d7e/contracts/WhitePaperInterestRateModel.sol#L17


# Vulnerability details

# [M-02] Incorrect `blocksPerYear` constant in `WhitepaperInterestRateModel`

## Location

https://github.com/code-423n4/2023-05-venus/blob/8be784ed9752b80e6f1b8b781e2e6251748d0d7e/contracts/WhitePaperInterestRateModel.sol#L17

## Impact

The interest rate per block is **5x** greater than it's intended to be for markets that use the Whitepaper interest rate model.

## Proof of Concept

The `WhitePaperInterestRateModel` contract is forked from Compound Finance, which was designed to be deployed on Ethereum Mainnet. The `blocksPerYear` constant inside the contract is used to calculate the interest rate of the market on a per-block basis and is set to **2102400**, which assumes that there are 365 days a year and that the block-time is **15 seconds**.

However, Venus Protocol is deployed on BNB chain, which has a block-time of only **3 seconds**. This results in the interest rate per block on the BNB chain to be **5x** greater than intended.

Both `baseRatePerBlock` and `multiplierPerBlock` are affected and are **5x** the value they should be. This also implies that the pool's interest rate is also 5 times more sensitive to utilization rate changes than intended. It is impossible for the market to arbitrage and adjust the interest rate back to the intended rate as seen in the PoC graph below. It's likely that arbitrageurs will deposit as much collateral as possible to take advantage of the high supply rate, leading to a utilization ratio close to 0.

The following Python script plots the `WhitePaperInterestRateModel` curves for a 15 second and a 3 second block time.

```python
import matplotlib.pyplot as plt

# Constants
BASE = 1e18

# Solidity functions converted to Python functions
def utilization_rate(cash, borrows, reserves):
    if borrows == 0:
        return 0
    return (borrows * BASE) / (cash + borrows - reserves)

def get_borrow_rate(ur, base_rate_per_block, multiplier_per_block):
    return ((ur * multiplier_per_block) / BASE) + base_rate_per_block

def generate_data_points(base_rate_per_year, multiplier_per_year, blocks_per_year, cash, borrows, reserves):
    base_rate_per_block = base_rate_per_year / blocks_per_year
    multiplier_per_block = multiplier_per_year / blocks_per_year

    utilization_rates = [i / 100 for i in range(101)]
    borrow_rates = [get_borrow_rate(ur * BASE, base_rate_per_block, multiplier_per_block) for ur in utilization_rates]

    return utilization_rates, borrow_rates

# User inputs
base_rate_per_year = 5e16 # 5%
multiplier_per_year = 1e16 # 1%
blocks_per_year1 = 2102400 # 15 second block-time
blocks_per_year2 = 10512000 # 3 second block-time

# Example values for cash, borrows, and reserves
cash = 1e18
borrows = 5e18
reserves = 0.1e18

# Generate data points for both curves
utilization_rates1, borrow_rates1 = generate_data_points(base_rate_per_year, multiplier_per_year, blocks_per_year1, cash, borrows, reserves)
utilization_rates2, borrow_rates2 = generate_data_points(base_rate_per_year, multiplier_per_year, blocks_per_year2, cash, borrows, reserves)

# Plot both curves on the same plot with a key
plt.plot(utilization_rates1, borrow_rates1, label=f"Blocks per year: {blocks_per_year1}")
plt.plot(utilization_rates2, borrow_rates2, label=f"Blocks per year: {blocks_per_year2}")
plt.xlabel("Utilization Rate")
plt.ylabel("Borrow Rate")
plt.title("Interest Rate Curves")
plt.legend()
plt.show()
```

Result:

![Interest Rate Per Block](https://i.imgur.com/0rotxUn.png)

As seen above, the borrow rate curves are different and do not intersect. Hence, it's impossible via arbitrage for market participants to adjust the rate back to its intended value.

## Tools Used

Manual review

## Recommended Mitigation Steps

Fix the `blocksPerYear` constant so that it accurately describes the number of blocks a year on BNB chain, which has a block-time of 15 seconds. The correct value is **10512000**.

```math
\begin{aligned}
\text{blocksPerYear} &= \frac{\text{secondsInAYear}}{\text{blockTime}} \\
&= \frac{365 \times 24 \times 60 \times 60}{3} \\
&= 10{,}512{,}000
\end{aligned}
```

```diff
@@ -14,7 +14,7 @@ contract WhitePaperInterestRateModel is InterestRateModel {
     /**
      * @notice The approximate number of blocks per year that is assumed by the interest rate model
      */
-    uint256 public constant blocksPerYear = 2102400;
+    uint256 public constant blocksPerYear = 10512000;

     /**
      * @notice The multiplier of utilization rate that gives the slope of the interest rate
```






## Assessed type

Other